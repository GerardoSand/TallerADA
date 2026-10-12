# TallerADA
```
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>

// ============================================================================
// CREDENCIALES DE TU RED WI-FI
// ============================================================================
const char* ssid     = "S22 Ultra de Gerardo";
const char* password = "gato1234";

// ======================================================
// CONFIGURACIÓN MQTT
// ======================================================
const char* mqtt_server = "10.211.40.136";
const int   mqtt_port   = 1883;

const char* TOPIC_TEMP = "clima/esp32/temperatura";
const char* TOPIC_HUM  = "clima/esp32/humedad";
const char* TOPIC_LUZ  = "clima/esp32/luz";

// ======================================================
// CONFIGURACIÓN DE SENSORES
// ======================================================
#define DHTPIN  4         
#define DHTTYPE DHT11  
#define LDR_PIN 34

DHT dht(DHTPIN, DHTTYPE);

WiFiClient espClient;
PubSubClient client(espClient);

const long INTERVALO_ENVIO = 2000;
unsigned long ultimoEnvio = 0;

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Conectando a Wi-Fi: ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\n[OK] Conectado a Wi-Fi.");
  Serial.print("IP asignada al ESP32: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Intentando conectar con Mosquitto (");
    Serial.print(mqtt_server);
    Serial.print(")... ");

    String clientId = "ESP32-";
    clientId += String(
      (uint32_t)(ESP.getEfuseMac() & 0xFFFFFFFF),
      HEX
    );

    if (client.connect(clientId.c_str())) {
      Serial.println("[OK] Conectado al Broker MQTT!");
    } else {
      Serial.print("[FALLO] rc=");
      Serial.print(client.state());
      Serial.println(" -> Reintentando en 3 segundos...");
      delay(3000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(100);

  // Iniciar sensor DHT11
  dht.begin();

  // Conectar a la red
  setup_wifi();

  // Configurar servidor MQTT
  client.setServer(mqtt_server, mqtt_port);
}

// ======================================================
// LEER Y PUBLICAR SENSORES
// ======================================================
void loop() {
  
  // Si se pierde MQTT, volver a conectarse.
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long EnvioActual = millis();
  if (EnvioActual - ultimoEnvio >= INTERVALO_ENVIO) {
    ultimoEnvio = EnvioActual;

    float humedad = dht.readHumidity();
    float temperatura = dht.readTemperature(); 
    int valorLdr = analogRead(LDR_PIN);

    // Verificar si el DHT11 respondi correctamente
    if (isnan(humedad) || isnan(temperatura)) {
      Serial.println("Error al leer el sensor DHT11!");
    } else {
      // publicacion de temperatura
      char tempStr[8];
      dtostrf(temperatura, 1, 1, tempStr);
      client.publish(TOPIC_TEMP, tempStr);

      char humStr[8];
      dtostrf(humedad, 1, 1, humStr);
      client.publish(TOPIC_HUM, humStr);

      // impresion de comprobacion
      Serial.print("Temp: ");
      Serial.print(tempStr);
      Serial.print(" °C | Hum: ");
      Serial.print(humStr);
      Serial.print(" % | ");
    }

    float voltaje = valorLdr * (3.3 / 4095.0);

    // Publicar Nivel de Luz
    char ldrStr[8];
    dtostrf(voltaje, 1, 2, ldrStr);
    client.publish(TOPIC_LUZ, ldrStr);

    // comprobar los valores de luz
    Serial.print("ADC: ");
    Serial.print(valorLdr);
    Serial.print(" | Voltaje: ");
    Serial.print(voltaje, 2);
    Serial.println(" V");
  }
}
```
