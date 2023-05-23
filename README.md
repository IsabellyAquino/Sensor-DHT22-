# Sensor-DHT22-
Projeto de teste e desenvolvimento com o Sensor DHT22 e ESP32 para aula de IOT-SENAI com a API  ThingSpeak.

(na ultima etapa fizemos conexão com hiveMQ e document desenvolvido pelo professor para converter json em MQTT)

SEGUE CÓDIGO COMENTADO PARA MELHOR ENTENDIMENTO DO PASSO A PASSO 



//inclui as bibliotecas
#include <DHTesp.h>
#include "WiFi.h"
#include "HTTPClient.h"
#include "PubSubClient.h"
// define pino dos Leds 1 e 2
#define LEDPIN1 14
#define LEDPIN2 12

// define pino do sensor
#define DHT_PIN 15

WiFiClient espClient;

//variáveis temp e umidade inicializam com zero
int temperatura = 0;
int umidade = 0;

//define objeto sensor
DHTesp dhtSensor;

//define objeto mqttclient
PubSubClient mqttClient(espClient);

// define variaveis de acesso do wi-fi padrão,
const char* ssid = "Wokwi-GUEST";
const char* password = "";


void setup() {
  Serial.begin(115200);

  pinMode(14, OUTPUT);
  pinMode(12, OUTPUT);

  //inicializando o sensor
  Serial.begin(115200);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  dhtSensor.getPin();
  delay(100);

  //inicializando a conexão com o Wi-fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  //imprime na tela mensagem de confirmação da conexão de wifi
  Serial.println("");
  Serial.println("WiFi conectado!");
  Serial.println(WiFi.localIP());

  //configuarando o mqtt broker
  Serial.println("Conectando ao Broker Mqtt");
  mqttClient.setServer("broker.hivemq.com", 1883);

  //conectando e garantindo com a função random que o IDClient seja único 
  String clientId = "esp32Des_senai-" + String(random(0xffff), HEX);
  boolean isConected = mqttClient.connect(clientId.c_str());

  //validação para que enquanto o isconected n for verdadeiro se repita até se conectar
  while(!isConected){
    clientId = "esp32Des_senai-" + String(random(0xffff), HEX);
    isConected = mqttClient.connect(clientId.c_str());
    Serial.println(".");
    }
  Serial.println("");
  Serial.println("Broker conectado!");
}

void loop() {
  TempAndHumidity data = dhtSensor.getTempAndHumidity();

  //Variáveis passam a receber valor do sensor
  float umidade = data.humidity;
  float temperatura = data.temperature;

  //Se temperatura for maior ou igual 35 LED1 acende e imprime mensagem de alerta
  if (temperatura >= 35) {
    digitalWrite(14, HIGH);
    Serial.println("ATENÇÃO: A temperatura ultrapassa 35°C!");
    Serial.println("TEMPERATURA: " + String(temperatura) + "°C" );
  } else {
    digitalWrite(14, LOW);
    Serial.println("TEMPERATURA: " + String(temperatura) + "°C" );
  }

  //Se a umidade for maior ou igual que 70% LED2 acende e imprime mensagem de alerta
  if (umidade >= 70) {
    digitalWrite(12, HIGH);
    Serial.println("ATENÇÃO: Umidade ultrapassa dos 70% no momento!");
    Serial.println("UMIDADE: " + String(umidade) + "%" );
  } else {
    digitalWrite(12, LOW);
    Serial.println("UMIDADE: " + String(umidade) + "%" );
  }
  Serial.println("___________________________________________________");

  //define objeto http
  HTTPClient http;

  //envia url da api para setar valores do sensor no momento
  String url = "https://api.thingspeak.com/update?api_key=CMTF41PLV6DMCUXA&field1=";

  //define valores do campo1 e campo2 recebendo do sensor e setando na url
  url = url + temperatura + "&field2=" + umidade;

  //inicializa o protocolo de comunicação HTTP
  http.begin(url);
  int httpCode = http.GET();
  String str;
  char valorStr[50];

  //imprime informações se status HTTP indicar valores entre 200 e 299
  Serial.println(httpCode);
  if (httpCode >= 200 && httpCode <= 299) {
    String payload = http.getString();
    Serial.println(payload);
    //se não imprime mensagem de alerta
  } else {
    Serial.println("Problema ao conectar API.");
  }

  // recebe na str uma string concatenando valores de temperatura e umidade convertidos para string
  str = "{\"temperatura\" : " + String(temperatura) + ", \"umidade\" : " + String(umidade) + " }";
  
  //guarda num array char todos os caracteres do str e guarda esse array em valorStr
  str.toCharArray(valorStr, str.length() + 1);

  //publica a string contendo o json para o mqtt
  mqttClient.publish("topic_on_senai", valorStr);


  //Define um intervalo de 5s para os dados capturados
  delay(5000);

}



