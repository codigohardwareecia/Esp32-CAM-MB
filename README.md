# Esp32-CAM-MB
Vc pode adquirir no Mercado Livre, o link é apenas um exemplo
https://www.mercadolivre.com.br/kit-c03-modulos-esp32-cam-mb-cmera-ov2640-e-antena-nota/p/MLB2069944466?pdp_filters=item_id:MLB3277415831

Adicione essa URL nas Additionals URls do Arduino IDE
https://espressif.github.io/arduino-esp32/package_esp32_index.json

Em boards instale o pacote ESP32 da ExpressIf

Selecine a board ESP32 Wrover Module

Conecte o a camera usando o flatcable ao móduo ESP32CAM

Conecte o módulo ESP32CAM ao ESP32CAMMB

Conecte o cabo USB e selecione a porta

Cole o código abaixo no Arduino

Altere os dados de Wifi e atribuia um endereço IP Fixo

```C++
#include "esp_camera.h"
#include <WiFi.h>

const char* ssid = "Seu Wifi";
const char* password = "123456789";

IPAddress local_IP(192, 168, 15, 92); // Este será o IP fixo da câmera
IPAddress gateway(192, 168, 15, 1);    // O IP do seu roteador
IPAddress subnet(255, 255, 255, 0);   // Máscara de sub-rede padrão
IPAddress primaryDNS(8, 8, 8, 8);     // DNS (opcional, mas bom manter)
IPAddress secondaryDNS(8, 8, 4, 4);

// Pinos de hardware exatos para o modelo AI Thinker
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

WiFiServer server(80);

void setup() {
  Serial.begin(115200);
  Serial.setDebugOutput(true);
  Serial.println();

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  
  // Verifica o PSRAM para alocar memória corretamente
  if(psramFound()){
    config.frame_size = FRAMESIZE_VGA; // Resolução 640x480
    config.jpeg_quality = 10;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_SVGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }

// Inicializa a câmera
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Falha na inicialização da câmera com o erro 0x%x", err);
    return;
  }

  // --- CÓDIGO NOVO PARA INVERTER A IMAGEM ---
  sensor_t * s = esp_camera_sensor_get();
  // Inverte verticalmente (0 = normal, 1 = invertido)
  s->set_vflip(s, 1); 
  // Espelha horizontalmente (0 = normal, 1 = espelhado)
  s->set_hmirror(s, 1);

  // Configura o IP Estático ANTES de conectar
  if (!WiFi.config(local_IP, gateway, subnet, primaryDNS, secondaryDNS)) {
    Serial.println("Falha ao configurar IP Estático");
  }

  // Conecta ao Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi conectado!");
  Serial.println("");
  Serial.println("WiFi conectado!");

  server.begin();
  Serial.print("Stream de vídeo pronto. Acesse: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  WiFiClient client = server.available();
  if (client) {
    Serial.println("Novo dispositivo conectado ao stream");
    String currentLine = "";
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        if (c == '\n') {
          if (currentLine.length() == 0) {
            // Cabeçalho HTTP para o stream MJPEG
            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:multipart/x-mixed-replace; boundary=frame");
            client.println();

            // Loop infinito enviando os frames
            while(client.connected()){
              camera_fb_t * fb = esp_camera_fb_get();
              if (!fb) {
                Serial.println("Falha na captura do frame");
                return;
              }
              client.printf("--frame\r\nContent-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n", fb->len);
              client.write(fb->buf, fb->len);
              client.println();
              esp_camera_fb_return(fb);
              delay(40); // Controle de framerate (~25fps)
            }
            break;
          } else {
            currentLine = "";
          }
        } else if (c != '\r') {
          currentLine += c;
        }
      }
    }
    client.stop();
    Serial.println("Dispositivo desconectado.");
  }
}
```
