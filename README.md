// Global Solution Edge
// Arthur Petrin (RM98735) Gabriel Mazza (RM99265)
// Video: https://www.youtube.com/watch?v=8Unsqq1YYNk

#include <WiFi.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <PubSubClient.h>

const char *ssid1 = "Wokwi-GUEST";   // Substitua pelo nome da sua rede
const char *password1 = "";          // Rede sem senha (criptografia aberta)
const char *mqtt_server = "46.17.108.113"; // Substitua pelo IP da sua VM

WiFiClient espClient;
PubSubClient client(espClient);

// Dados do dispositivo
const char *device_id = "hosp200";
const char *entity_name = "urn:ngsi-ld:hosp:200";
const char *entity_type = "hosp";
const char *protocol = "PDI-IoTA-UltraLight";
const char *transport = "MQTT";
const char *mqtt_topic = "hosp200"; // Tópico MQTT para publicar e se inscrever

// Pino onde o sensor DS18B20 está conectado
const int oneWireBus = 4; // Exemplo: pino D4

OneWire oneWire(oneWireBus);
DallasTemperature sensors(&oneWire);

bool temperaturePublished = false;  // Variável de controle

void setup() {
  Serial.begin(115200);

  // Conectar-se à rede WiFi
  WiFi.begin(ssid1, password1);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Conectando ao WiFi...");
  }
  Serial.println("Conectado ao WiFi");

  // Configuração do cliente MQTT
  client.setServer(mqtt_server, 1883);  // 1883 é a porta padrão para MQTT

  // Inicializar o sensor de temperatura
  sensors.begin();
}

void loop() {
  // Reconectar ao MQTT, se necessário
  if (!client.connected()) {
    reconnect();
  }

  // Código principal do loop
  client.loop(); // Adicionado para processar mensagens MQTT

  // Leia a temperatura do sensor DS18B20 apenas se ainda não foi publicada
  if (!temperaturePublished) {
    float temperature = readTemperature();
  
    // Publique a temperatura no tópico MQTT
    publishTemperature(temperature);

    // Defina a variável de controle como true para evitar novas publicações
    temperaturePublished = true;
  }

  // Adicione a lógica específica do seu aplicativo aqui
}

void reconnect() {
  // Loop até que estejamos reconectados
  while (!client.connected()) {
    Serial.println("Tentando reconectar ao MQTT...");
    if (client.connect("hosp200")) {
      Serial.println("Conectado ao servidor MQTT");

      // Assine os tópicos necessários aqui
      client.subscribe(mqtt_topic);
      
      // Publica os atributos iniciais do dispositivo
      publishAttributes();
    } else {
      Serial.print("Falha na conexão. Retorno = ");
      Serial.println(client.state());
      delay(5000);  // Aguarde 5 segundos antes de tentar novamente
    }
  }
}

void publishAttributes() {
  // Publica os atributos iniciais do dispositivo no tópico "hosp200"
  client.publish(mqtt_topic, "state=off");
  client.publish(mqtt_topic, "temperature=0.0");
  client.publish(mqtt_topic, "luminosity=0");
}

float readTemperature() {
  sensors.requestTemperatures(); // Solicitar leitura da temperatura
  float temperature = sensors.getTempCByIndex(0); // Obter temperatura em graus Celsius
  Serial.print("Temperatura: ");
  Serial.println(temperature);
  return temperature;
}

void publishTemperature(float temperature) {
  String payload = "temperature=" + String(temperature);
  client.publish(mqtt_topic, payload.c_str());
}
