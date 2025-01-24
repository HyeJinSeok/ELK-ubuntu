# 🚀 ELK-ubuntu 

## 🖥️ 1. 전체 아키텍처 
- **Ubuntu 20.04 LTS** 가상머신 3대를 활용하여 **ELK 스택**(Logstash, Elasticsearch, Kibana) 구축

<p align="center">
  <img src="https://github.com/user-attachments/assets/55e91cc3-c835-4766-b049-9517706c432f" alt="Architecture Diagram">
</p>

### 1. VirtualBox
- 위의 모든 구성 요소(**Logstash**, **Elasticsearch**, **Kibana**)가 VirtualBox에서 실행
- 네트워크 설정
  - 각 가상머신의 네트워크 어댑터를 **브리지 어댑터**로 설정
  - 모든 가상머신이 **동일한 서브넷**에 연결되도록 구성하여 가상머신 간 원활한 통신을 보장

### 2. Windows Host
- 사용자 환경에서 **Kibana 웹 인터페이스**를 통해 데이터를 시각화하고 분석
1. 브라우저에서 가상머신의 Kibana가 사용하는 IP 주소와 포트(예: `http://<Kibana IP>:5601`)를 입력
2. Kibana 대시보드에 접속하여 데이터 시각화 및 분석 수행

## 🐧 2. ELK Stack on Ubuntu
### ✅ 1. Java 설치 확인 및 설치
- **Elasticsearch 7.16.3**은 **Java 8 이상**이 필요
```bash
# 1. Java 설치 확인
java -version

# 2. Java가 설치되어 있지 않다면 OpenJDK 11 설치
sudo apt update
sudo apt install -y openjdk-11-jdk
```

### ✅ 2. Elasticsearch 7.16.3 설치 및 설정
**1. Elasticsearch GPG 키 추가**
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```
**2. Elastic 패키지 저장소 추가**
```bash
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
```
**3. 패키지 리스트 업데이트 & Elasticsearch 설치**
```bash
sudo apt update
sudo apt install -y elasticsearch=7.16.3
```
**4. Elasticsearch 서비스 활성화**
```bash
# Elasticsearch 자동 시작 활성화
sudo systemctl enable elasticsearch

# Elasticsearch 시작
sudo systemctl start elasticsearch

# 정상적으로 실행 중인지 확인
sudo systemctl status elasticsearch
```
**5. Elasticsearch 설정 파일 변경**
- 1개의 Elasticsearch 노드만 실행시킬 것이기 때문에 단일 노드로 설정
```bash
sudo vi /etc/elasticsearch/elasticsearch.yml

cluster.name: my-cluster     # 클러스터 이름
node.name: my-node           # 노드 이름
discovery.type: single-node  # 단일 노드 모드 활성화

# 네트워크 설정
network.host: 0.0.0.0        # 외부에서도 접근 가능하게 설정
http.port: 9200              # HTTP 요청 포트
```
**6. Elasticsearch 재시작 및 연결 확인**
```bash
# Elasticsearch 재시작
sudo systemctl restart elasticsearch

# 상태 확인
sudo systemctl status elasticsearch

# 연결 확인
curl -X GET "http://localhost:9200/"
```
### ✅ 3. Logstash 7.16.3 설치 및 설정
**1. Logstash 패키지 다운로드 및 설치**
```bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-7.16.3-amd64.deb
sudo dpkg -i logstash-7.16.3-amd64.deb
```
**2. Logstash 설정 파일**
```bash
sudo vi /etc/logstash/logstash.yml

path.data: /var/lib/logstash
path.logs: /var/log/logstash
http.host: "0.0.0.0"   # 모든 인터페이스에서 수신 허용
```
**3. Logstash 구성 파일**
- 구성 파일 (`/etc/logstash/conf.d/logstash.conf`)를 생성하여 입력 및 출력 파이프라인을 설정
- 본 프로젝트에서는 Kibana에서 csv 데이터를 직접 import하여 사용
```bash
sudo vi /etc/logstash/conf.d/logstash.conf

# 파일 예제 - Filebeat 활용 시 
input {
    beats {
        port => 5044
    }
}
output {
    elasticsearch {
        hosts => ["http://<Elasticsearch_IP>:9200"]
        index => "logstash-%{+YYYY.MM.dd}"
    }
    stdout { codec => rubydebug }
}
```
**4. Logstash 시작 및 상태 확인**
```bash
sudo systemctl start logstash
sudo systemctl enable logstash
sudo systemctl status logstash
```
### ✅ 4. Kibana 7.16.3 설치 및 설정
**1. Kibana 패키지 다운로드 및 설치**
```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.16.3-amd64.deb
sudo dpkg -i kibana-7.16.3-amd64.deb
```
**2. Kibana 설정 파일 변경**
```bash
sudo vi /etc/kibana/kibana.yml

server.host: "0.0.0.0"     # 외부에서 접속 가능하도록 설정
elasticsearch.hosts: ["http://<Elasticsearch_IP>:9200"]
```
**3. Kibana 시작 및 상태 확인**
```bash
sudo systemctl start kibana
sudo systemctl enable kibana
sudo systemctl status kibana
```
### ✅ 5. Host 웹 브라우저에서 접속 확인
- Elasticsearch: 브라우저에서 http://<Elasticsearch_IP>:9200에 접속하여 Elasticsearch 상태 확인
- Kibana: 브라우저에서 http://<Kibana_IP>:5601에 접속하여 Kibana 대시보드 확인
- Logstash : Data import(index : carddata) 후 index 확인
  <p align="center">
    <img src="https://github.com/user-attachments/assets/f799b2b2-6c37-4a77-99ae-64531d4f6c49" alt="Architecture Diagram">
  </p>



## 🛠️ 3. ElasticSearch 7.16.3 버전 선택 이유
<br>

  | **항목**            | **8.x 버전**                                      | **7.x 버전**                                      |
  |---------------------|------------------------------------------------------------|------------------------------------------------------------|
  | **보안**       | TLS/SSL 암호화, 인증 설정 등 보안 기능 포함, 운영 환경에서 보안성 높음 | 보안 설정 없음, 상대적으로 간단하고 빠른 설정 가능            |
  | **설정 복잡도**     | 보안 설정을 추가해야 하므로 설정이 복잡하고 관리 어려울 수 있음 | 보안 설정 필요 없음, 설정 간단하여 학습용으로 용이            |
  | **성능 개선**       | 성능 및 기능 개선 많음, 하지만 학습에 복잡할 수 있음           | 성능 개선이 적지만 안정적이고 쉽게 학습 가능                   |
  | **학습 용이성**     | 보안 설정 필요하여 학습이 복잡해질 수 있음                     | 보안 설정 없이 바로 실습 가능, 빠르고 직관적인 학습 가능        |
  | **기존 API와 호환성** | 새로운 기능을 제공하지만, 기존 API와 호환성 문제 가능         | 기존 7버전 API와 설정이 그대로 유지되어 혼동 적음              |
  | **적합성**          | 운영 환경에서 보안 및 성능 면에서 유리하지만 학습에는 불편함     | 학습용으로 더 적합, 보안 설정 없이 빠르게 실습 가능              |

- 리눅스 설치 환경 학습을 목적으로, 보안 관련 복잡성이 적은 7.x 버전을 설치
-  7.16.x 버전은 Elasticsearch 7.x 시리즈에서 제공하는 최신 기능들을 포함
- 새로운 REST API 기능, 쿼리 성능 향상, 일부 새로운 플러그인 기능 등이 추가되어 실습 시 더욱 효율적으로 작업 가능
-  7.16.3은 7.x 시리즈에서 커뮤니티의 활발한 지원을 받는 마지막 주요 버전 
-  **7.16.3 버전은 보안 패치와 버그 수정이 완료되어 안정적이며, 최신 기능도 갖추고 있어 학습용으로 적합하면서도 시스템에 부담을 덜 수 있는 버전**
