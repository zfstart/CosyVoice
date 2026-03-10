# CosyVoice 部署与 Spring Boot 调用指南（TTS）

本文基于仓库内 `runtime/python/fastapi/server.py` 与 `runtime/python/fastapi/client.py` 的实现，说明如何部署 CosyVoice 服务，并在 Spring Boot 中调用 API 完成语音合成。

## 1. 推荐部署方式

仓库 README 已提供容器化部署命令，推荐直接使用 `runtime/python` 下的 Dockerfile 构建服务镜像。

```bash
cd runtime/python
docker build -t cosyvoice:v1.0 .

# SFT 模型示例（可替换 model_dir）
docker run -d --runtime=nvidia -p 50000:50000 cosyvoice:v1.0 \
  /bin/bash -c "cd /opt/CosyVoice/CosyVoice/runtime/python/fastapi \
  && python3 server.py --port 50000 --model_dir iic/CosyVoice-300M && sleep infinity"
```

启动后，FastAPI 默认监听 `0.0.0.0:<port>`。

## 2. FastAPI 接口说明（与 server.py 对齐）

- `POST /inference_sft`
  - 表单字段：`tts_text`, `spk_id`
- `POST /inference_zero_shot`
  - 表单字段：`tts_text`, `prompt_text`, 文件字段：`prompt_wav`
- `POST /inference_cross_lingual`
  - 表单字段：`tts_text`, 文件字段：`prompt_wav`
- `POST /inference_instruct`
  - 表单字段：`tts_text`, `spk_id`, `instruct_text`
- `POST /inference_instruct2`
  - 表单字段：`tts_text`, `instruct_text`, 文件字段：`prompt_wav`

> 注意：接口返回是流式二进制音频块，服务端输出的是 **16-bit PCM 原始字节流**（不是带 WAV 头的完整文件）。

## 3. Spring Boot 调用示例

以下示例使用 `WebClient`，建议用于流式读取。

### 3.1 Maven 依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### 3.2 SFT 模式调用（文本转语音）

```java
@Service
public class CosyVoiceClient {

    private final WebClient webClient;

    public CosyVoiceClient(@Value("${cosyvoice.base-url}") String baseUrl) {
        this.webClient = WebClient.builder()
                .baseUrl(baseUrl) // 例如 http://127.0.0.1:50000
                .build();
    }

    public byte[] ttsSft(String text, String spkId) {
        return webClient.post()
                .uri("/inference_sft")
                .contentType(MediaType.MULTIPART_FORM_DATA)
                .body(BodyInserters
                        .fromMultipartData("tts_text", text)
                        .with("spk_id", spkId))
                .retrieve()
                .bodyToFlux(DataBuffer.class)
                .map(dataBuffer -> {
                    byte[] bytes = new byte[dataBuffer.readableByteCount()];
                    dataBuffer.read(bytes);
                    DataBufferUtils.release(dataBuffer);
                    return bytes;
                })
                .reduce(new ByteArrayOutputStream(), (bos, chunk) -> {
                    try {
                        bos.write(chunk);
                    } catch (IOException e) {
                        throw new UncheckedIOException(e);
                    }
                    return bos;
                })
                .map(ByteArrayOutputStream::toByteArray)
                .block();
    }
}
```

### 3.3 Zero-shot / Cross-lingual 文件上传示例

```java
public byte[] ttsZeroShot(String text, String promptText, Resource promptWav) {
    MultipartBodyBuilder builder = new MultipartBodyBuilder();
    builder.part("tts_text", text);
    builder.part("prompt_text", promptText);
    builder.part("prompt_wav", promptWav);

    return webClient.post()
            .uri("/inference_zero_shot")
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .body(BodyInserters.fromMultipartData(builder.build()))
            .retrieve()
            .bodyToFlux(DataBuffer.class)
            .map(db -> {
                byte[] bytes = new byte[db.readableByteCount()];
                db.read(bytes);
                DataBufferUtils.release(db);
                return bytes;
            })
            .reduce(new ByteArrayOutputStream(), (bos, chunk) -> {
                try {
                    bos.write(chunk);
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
                return bos;
            })
            .map(ByteArrayOutputStream::toByteArray)
            .block();
}
```

## 4. 把返回 PCM 保存为标准 WAV

由于服务端返回的是裸 PCM，建议在 Java 侧补上 WAV Header 后再存储/播放。

```java
public byte[] pcm16ToWav(byte[] pcmData, int sampleRate, int channels, int bitsPerSample) throws IOException {
    int byteRate = sampleRate * channels * bitsPerSample / 8;
    int blockAlign = channels * bitsPerSample / 8;
    int dataSize = pcmData.length;
    int chunkSize = 36 + dataSize;

    ByteArrayOutputStream out = new ByteArrayOutputStream();
    DataOutputStream dos = new DataOutputStream(out);

    dos.writeBytes("RIFF");
    dos.writeInt(Integer.reverseBytes(chunkSize));
    dos.writeBytes("WAVE");
    dos.writeBytes("fmt ");
    dos.writeInt(Integer.reverseBytes(16));
    dos.writeShort(Short.reverseBytes((short) 1));
    dos.writeShort(Short.reverseBytes((short) channels));
    dos.writeInt(Integer.reverseBytes(sampleRate));
    dos.writeInt(Integer.reverseBytes(byteRate));
    dos.writeShort(Short.reverseBytes((short) blockAlign));
    dos.writeShort(Short.reverseBytes((short) bitsPerSample));
    dos.writeBytes("data");
    dos.writeInt(Integer.reverseBytes(dataSize));
    dos.write(pcmData);
    dos.flush();

    return out.toByteArray();
}
```

推荐参数：`sampleRate=22050`, `channels=1`, `bitsPerSample=16`。

## 5. 在业务中落地建议

1. **网关隔离**：Spring Boot 不直接暴露 CosyVoice 容器端口，对外只暴露业务 API。
2. **异步化**：长文本合成建议走消息队列 + 回调/轮询状态。
3. **超时与熔断**：为 WebClient 配置连接/读取超时和重试策略。
4. **资源控制**：GPU 服务建议按并发做限流，避免 OOM。
5. **模型选择**：
   - 常规克隆音色：`/inference_zero_shot`
   - 跨语种迁移：`/inference_cross_lingual`
   - 角色/风格指令：`/inference_instruct` 或 `/inference_instruct2`

## 6. 可选方案：gRPC（更适合 Java）

仓库也提供 gRPC 接口（`runtime/python/grpc/cosyvoice.proto`）。Java 端可直接用 proto 生成 stub，调用 `CosyVoice.Inference` 的服务端流式响应，在高并发下通常比 HTTP 表单上传更稳定。


## 7. 最小可运行 Spring Boot 模板（可直接粘贴）

下面给你一套最小结构，默认调用 `/inference_sft`，并把 CosyVoice 返回的 PCM 包装成 WAV 再返回给前端下载。

### 7.1 `application.yml`

```yaml
server:
  port: 8080

cosyvoice:
  base-url: http://127.0.0.1:50000
  sample-rate: 22050
```

### 7.2 `CosyVoiceProperties.java`

```java
package com.example.tts.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "cosyvoice")
public class CosyVoiceProperties {
    private String baseUrl;
    private int sampleRate = 22050;

    public String getBaseUrl() {
        return baseUrl;
    }

    public void setBaseUrl(String baseUrl) {
        this.baseUrl = baseUrl;
    }

    public int getSampleRate() {
        return sampleRate;
    }

    public void setSampleRate(int sampleRate) {
        this.sampleRate = sampleRate;
    }
}
```

### 7.3 `CosyVoiceConfig.java`

```java
package com.example.tts.config;

import io.netty.channel.ChannelOption;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;

@Configuration
@EnableConfigurationProperties(CosyVoiceProperties.class)
public class CosyVoiceConfig {

    @Bean
    public WebClient cosyVoiceWebClient(CosyVoiceProperties properties) {
        HttpClient httpClient = HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
                .responseTimeout(Duration.ofSeconds(120));

        return WebClient.builder()
                .baseUrl(properties.getBaseUrl())
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}
```

### 7.4 `TtsRequest.java`

```java
package com.example.tts.dto;

public class TtsRequest {
    private String text;
    private String spkId;

    public String getText() {
        return text;
    }

    public void setText(String text) {
        this.text = text;
    }

    public String getSpkId() {
        return spkId;
    }

    public void setSpkId(String spkId) {
        this.spkId = spkId;
    }
}
```

### 7.5 `CosyVoiceService.java`

```java
package com.example.tts.service;

import com.example.tts.config.CosyVoiceProperties;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.client.WebClient;

import java.io.ByteArrayOutputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.UncheckedIOException;

@Service
public class CosyVoiceService {

    private final WebClient webClient;
    private final CosyVoiceProperties properties;

    public CosyVoiceService(WebClient cosyVoiceWebClient, CosyVoiceProperties properties) {
        this.webClient = cosyVoiceWebClient;
        this.properties = properties;
    }

    public byte[] ttsSftToWav(String text, String spkId) {
        byte[] pcm = webClient.post()
                .uri("/inference_sft")
                .contentType(MediaType.MULTIPART_FORM_DATA)
                .body(BodyInserters
                        .fromMultipartData("tts_text", text)
                        .with("spk_id", spkId))
                .retrieve()
                .bodyToFlux(DataBuffer.class)
                .map(db -> {
                    byte[] bytes = new byte[db.readableByteCount()];
                    db.read(bytes);
                    DataBufferUtils.release(db);
                    return bytes;
                })
                .reduce(new ByteArrayOutputStream(), (bos, chunk) -> {
                    try {
                        bos.write(chunk);
                    } catch (IOException e) {
                        throw new UncheckedIOException(e);
                    }
                    return bos;
                })
                .map(ByteArrayOutputStream::toByteArray)
                .block();

        if (pcm == null || pcm.length == 0) {
            throw new IllegalStateException("CosyVoice 返回空音频");
        }

        try {
            return pcm16ToWav(pcm, properties.getSampleRate(), 1, 16);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private byte[] pcm16ToWav(byte[] pcmData, int sampleRate, int channels, int bitsPerSample) throws IOException {
        int byteRate = sampleRate * channels * bitsPerSample / 8;
        int blockAlign = channels * bitsPerSample / 8;
        int dataSize = pcmData.length;
        int chunkSize = 36 + dataSize;

        ByteArrayOutputStream out = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(out);

        dos.writeBytes("RIFF");
        dos.writeInt(Integer.reverseBytes(chunkSize));
        dos.writeBytes("WAVE");
        dos.writeBytes("fmt ");
        dos.writeInt(Integer.reverseBytes(16));
        dos.writeShort(Short.reverseBytes((short) 1));
        dos.writeShort(Short.reverseBytes((short) channels));
        dos.writeInt(Integer.reverseBytes(sampleRate));
        dos.writeInt(Integer.reverseBytes(byteRate));
        dos.writeShort(Short.reverseBytes((short) blockAlign));
        dos.writeShort(Short.reverseBytes((short) bitsPerSample));
        dos.writeBytes("data");
        dos.writeInt(Integer.reverseBytes(dataSize));
        dos.write(pcmData);
        dos.flush();

        return out.toByteArray();
    }
}
```

### 7.6 `TtsController.java`

```java
package com.example.tts.controller;

import com.example.tts.dto.TtsRequest;
import com.example.tts.service.CosyVoiceService;
import org.springframework.http.ContentDisposition;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/tts")
public class TtsController {

    private final CosyVoiceService cosyVoiceService;

    public TtsController(CosyVoiceService cosyVoiceService) {
        this.cosyVoiceService = cosyVoiceService;
    }

    @PostMapping(value = "/sft", produces = "audio/wav")
    public ResponseEntity<byte[]> sft(@RequestBody TtsRequest request) {
        String text = request.getText();
        String spkId = request.getSpkId();

        if (text == null || text.isBlank()) {
            return ResponseEntity.badRequest().build();
        }
        if (spkId == null || spkId.isBlank()) {
            spkId = "中文女";
        }

        byte[] wav = cosyVoiceService.ttsSftToWav(text, spkId);

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        ContentDisposition.attachment().filename("tts.wav").build().toString())
                .contentType(MediaType.parseMediaType("audio/wav"))
                .body(wav);
    }
}
```

### 7.7 `pom.xml` 关键依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

> 说明：如果你的服务仅作为 API 网关转发并使用 `WebClient`，保留 `webflux` 即可；若项目已有 MVC 控制器，保留 `web` + `webflux` 组合也可正常工作。

## 8. 调试命令（先验证链路）

```bash
curl -X POST 'http://127.0.0.1:8080/api/tts/sft' \
  -H 'Content-Type: application/json' \
  -d '{"text":"你好，我是CosyVoice。","spkId":"中文女"}' \
  --output tts.wav
```

如果 `tts.wav` 可播放，说明 Spring Boot -> CosyVoice 的端到端链路已打通。
