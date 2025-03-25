Este desafio tem como objetivo o planejamento e a documentação arquitetural e detalhamento, a estrutura precisa ser escalável e resiliente para geração de **XLSX e PDF**, 
utilizando **mensageria** para um processamento assíncrono e garantindo a **idempotência**. Isso envolve diversas decisões arquiteturais. 

### **Justificativa Técnica**  

#### ** Escolha da Tecnologia de Mensageria**  
Optei pelo **RabbitMQ** devido à sua confiabilidade no processamento de filas de trabalho, suporte a **mensagens persistentes**, roteamento flexível e facilidade de integração com diversos serviços. Diferente do Kafka, que é mais indicado para streaming de eventos e processamento em larga escala, o RabbitMQ é mais eficiente para **distribuição de tarefas assíncronas**, garantindo que cada mensagem seja processada exatamente uma vez por um consumidor. Além disso, o suporte a **acknowledgments e Dead Letter Queue (DLQ)** possibilita melhor controle de falhas.  

#### ** Estratégia de Escalabilidade**  
Para garantir alta disponibilidade e desempenho, adotei uma **estratégia de escalabilidade horizontal**, onde múltiplos **pods de consumidores** são distribuídos dinamicamente conforme a demanda. O **auto scaling** ajusta a quantidade de instâncias com base no número de mensagens na fila. Caso a carga aumente, novos consumidores são adicionados automaticamente, evitando gargalos. Além disso, o RabbitMQ pode ser configurado em **cluster**, permitindo balanceamento de carga e tolerância a falhas.  

#### ** Como Garantir Idempotência e Evitar Reprocessamento**  
A idempotência é essencial para evitar a geração duplicada de relatórios. Para isso, cada requisição recebe um **UUID único** baseado nos parâmetros enviados. Antes de processar a mensagem, verificamos no **Redis** ou no **banco de dados** se a solicitação já foi concluída. Caso já exista um resultado para aquele ID, a mensagem é descartada. Além disso, usamos **mensagens persistentes** no RabbitMQ, garantindo que um job não seja perdido ou executado mais de uma vez por erro de comunicação.  

#### ** Como Lidar com Falhas e Retries**  
Falhas podem ocorrer por diversos motivos, como indisponibilidade do banco ou erro na geração do relatório. Para evitar perda de mensagens e garantir resiliência, implementamos **retries exponenciais**, onde a mensagem é reprocessada após um tempo crescente a cada falha. Se a mensagem falhar repetidamente após um limite definido, ela é enviada para uma **Dead Letter Queue (DLQ)**, onde pode ser analisada e corrigida manualmente ou por processos automatizados.  

#### ** Uso de Banco de Dados e Cache**  
O **PostgreSQL** é utilizado para armazenar **metadados** das solicitações, como status, tempo de geração e referência ao arquivo gerado. Para otimizar o desempenho e reduzir consultas repetitivas ao banco, foi utlizado o **Redis** como cache para armazenar estados temporários e evitar reprocessamento. O Redis também é usado para gerenciar **locks distribuídos**, garantindo que apenas um processo finalize uma solicitação específica.  

A baixo, descrevo os detalho desta proposta de projeto.

---

## **1. Requisitos do Sistema**
Antes de definir a arquitetura, é essencial entender os requisitos:
- **Escalabilidade:** O sistema deve lidar com alta demanda, distribuindo a carga.
- **Resiliência:** Falhas devem ser toleradas, garantindo a continuidade do serviço.
- **Processamento Assíncrono:** As requisições de geração não devem bloquear o usuário.
- **Idempotência:** A mesma requisição deve gerar o mesmo resultado, evitando duplicações.
- **Monitoramento e Observabilidade:** Logs, métricas e alertas para identificar falhas.

---

## **2. Arquitetura do Sistema**

A arquitetura proposta é baseada em **microserviços** e utilizará um **message broker** para desacoplar a geração de relatórios da requisição do usuário.

### **2.1 Fluxo do Sistema**
1. **Recebimento da Solicitação**  
   - Um **serviço HTTP (REST ou GraphQL)** recebe a solicitação do usuário para gerar um XLSX ou PDF.
   - A requisição é validada e um **ID único (UUID ou hash)** é gerado para garantir idempotência.

2. **Publicação na Fila**  
   - O serviço publica uma mensagem contendo os detalhes da solicitação em uma **fila de mensagens** (RabbitMQ, Kafka, SQS, etc.).
   - A mensagem pode conter:
     - ID da solicitação
     - Tipo de arquivo (XLSX/PDF)
     - Parâmetros necessários (dados, formato, metadados)

3. **Processamento Assíncrono**  
   - Um **worker (consumidor)** lê a mensagem da fila e processa a geração do relatório.
   - O worker:
     - Recupera os dados necessários.
     - Gera o arquivo (usando Pandas para XLSX e ReportLab para PDF, por exemplo).
     - Armazena o arquivo em um **bucket de storage** (S3, GCS, MinIO, etc.).
     - Atualiza o status no banco de dados.

4. **Disponibilização do Relatório**  
   - O usuário pode consultar o status da geração por meio de um **endpoint REST** ou via **notificação (Webhook, WebSocket, email)**.
   - O arquivo pode ser baixado diretamente do **storage**.

---

## **3. Tecnologias**
### **3.1 Processamento Assíncrono (Mensageria)**
- **RabbitMQ**: Ideal para filas tradicionais com garantia de entrega e reprocessamento.
- **Apache Kafka**: Melhor para cenários com alto volume de mensagens e necessidade de reprocessamento.
- **Amazon SQS**: Gerenciado, boa opção para ambientes AWS.

### **3.2 Geração de Arquivos**
- **XLSX**: `Pandas`, `openpyxl`, `xlsxwriter`
- **PDF**: `ReportLab`, `WeasyPrint`, `wkhtmltopdf`

### **3.3 Armazenamento de Arquivos**
- **Amazon S3**
- **Google Cloud Storage**
- **MinIO (alternativa self-hosted)**

### **3.4 Banco de Dados**
- **PostgreSQL/MySQL**: Para armazenar metadados dos relatórios.
- **Redis**: Para cache de requisições e controle de idempotência.

---

## **4. Garantia de Idempotência**
A idempotência garante que, se uma requisição for enviada múltiplas vezes, apenas um relatório será gerado.

### **4.1 Estratégias**
- **Uso de um ID único**: Cada requisição recebe um `UUID` ou `hash dos parâmetros`. Antes de processar, o worker verifica se o ID já foi processado.
- **Banco de dados (Chave Única)**: Criar uma chave única no banco (`id_requisicao`) e rejeitar duplicatas.
- **Armazenamento de Estado**: Usar Redis para rastrear requisições já processadas.

---

## **5. Resiliência e Escalabilidade**
Para garantir a robustez do sistema:

### **5.1 Resiliência**
- **Retries com Exponential Backoff**: Evita sobrecarregar o sistema ao tentar processar falhas imediatamente.
- **Dead Letter Queue (DLQ)**: Mensagens que falham após várias tentativas são enviadas para uma fila separada para análise manual.
- **Circuit Breaker**: Protege contra falhas em serviços externos (exemplo: *Netflix Hystrix*).

### **5.2 Escalabilidade**
- **Escalar horizontalmente os workers**: Dependendo da carga, novos consumidores podem ser iniciados automaticamente.
- **Auto Scaling para o armazenamento**: Se usar um serviço de cloud, configurar escalabilidade para storage (S3/GCS).
- **Shard de banco de dados**: Para distribuir a carga conforme necessário.

---

## **6. Monitoramento e Observabilidade**
Para manter o controle da operação:
- **Logs centralizados**: Usar ELK Stack (Elasticsearch, Logstash, Kibana) ou Loki/Grafana.
- **Métricas e Alertas**: Prometheus + Grafana para monitorar tempo de geração, filas, erros.
- **Tracing distribuído**: OpenTelemetry ou Jaeger para rastrear chamadas entre serviços.

---

## **7. Benefícios da Arquitetura**
✅ **Escalabilidade:** A fila desacopla a solicitação da geração, permitindo escalar os consumidores.  
✅ **Resiliência:** O uso de DLQ e retries melhora a tolerância a falhas.  
✅ **Processamento Assíncrono:** O usuário não precisa esperar a geração do arquivo.  
✅ **Idempotência:** Evita duplicações e desperdício de processamento.  
✅ **Monitoramento:** Logs e métricas facilitam a detecção de problemas.

---

## **Conclusão**
Essa arquitetura é um exemplo para uma solução eficiente e robusta para geração de **XLSX e PDF** em larga escala. 
