# Docker 이미지 최적화 <br>
### Docker 이미지 최적화를 통한 빌드시간과 용량 개선 프로젝트
<br>

Spring Boot 애플리케이션을 기반으로 Docker 이미지의 빌드 시간과 이미지 크기를 측정하고 **3개의 최적화** 전략을 적용하여 성능 변화를 분석합니다. <br>
<br>단순히 Dockerfile을 작성하고 실행하는 수준을 넘어서, **실무 서비스 환경**을 가정하고 **CI/CD적 반복적**인 빌드와 배포 과정에서 발생할 수 있는 성능 문제를 수치적으로 확인하는 데 목적이 있습니다.

특히 **베이스 이미지 선택, 이미지 구조, Docker layer cache**과 같은 요소들이 **빌드 시간과 이미지 크기**에 어떤 영향을 주는지 비교하고, 이를 통해<br>
**효율적인 이미지 구성 방식**에 대한 이해를 높이는 데 초점을 둡니다.<br>


## 👥 팀원 소개
|  |  |  
| :---: | :---: |
| <img src="https://github.com/seajihey.png" width="180px"> | <img src="https://github.com/dbehdrbs0806.png" width="180px">  |
| [서지혜](https://github.com/seajihey) | [유동균](https://github.com/dbehdrbs0806)  |

<br><hr>
## [ 📌 실습 목적  ]
<br>

* **실무 서비스 수준**의 프로젝트 기반 Docker 빌드 성능 측정
* **Docker layer cache**에 따른 리빌드 성능 변화 확인
* **Multi-stage, Jlink**의 최적화 전략 적용
* 최적화 전과 후의 **빌드 시간 및 이미지 크기** 비교
<br>


## [ 📁 사용 서비스 선정 이유 ]

<br>

단순 테스트용 코드가 아닌,
실무 서비스 구조를 가진 프로젝트를 사용해야 **의미 있는 최적화 비교**가 가능합니다.

따라서 아래 GitHub 프로젝트를 사용했습니다.

* [https://github.com/dockersamples/spring-petclinic-docker.git](https://github.com/dockersamples/spring-petclinic-docker.git)

선정 이유는 아래와 같습니다.

1. Spring Boot 기반 실무 서비스 구조
2. **다양한 의존성**을 통한 현실적인 이미지 크기 발생 가능
3. 단순 샘플 대비 **최적화 효과**를 명확히 비교 가능

<br>

---
# 경량화 전 실습 과정

## 1️⃣ 깃허브에서 프로젝트 클론

linux 서버에서 실습을 진행하며, Docker 성능 측정을 위해 선정한 Spring Boot 프로젝트를 ``clone``합니다. <br>이후 해당 디렉터리로 이동하여 Dockerfile과 애플리케이션 구조를 확인합니다.

```bash
mkdir DockerOptimizerLab
cd DockerOptimizerLab
git clone https://github.com/dockersamples/spring-petclinic-docker.git
cd spring-petclinic-docker
```

### 📄 원본 Dockerfile 확인

프로젝트에 포함된 기본 Dockerfile을 확인하여 현재 이미지 빌드 방식과 구조를 파악합니다. <br>
```dockerfile
FROM eclipse-temurin:21-jdk-jammy
 
WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:resolve

COPY src ./src

CMD ["./mvnw", "spring-boot:run"]
```
기본 Dockerfile은 **JDK 기반 이미지**를 사용하며, **Maven Wrapper를 통해 의존성을 다운로드**한 뒤 애플리케이션을 실행하는 구조입니다. <br>

이는 **dependency를 먼저 처리**한 후 **source를 복사**하는 방식으로 일부 캐시 활용이 가능하지만, 
빌드와 실행이 **동일 컨테이너**에서 이루어지고 <br>
**JDK 이미지를 그대로** 사용하기 때문에 이미지 크기와 빌드 효율 측면에서 개선 여지가 존재합니다.
<br>

---

## 2️⃣ 1차 실험: 원본 Dockerfile 빌드 시간 측정

초기 상태에서의 기준 데이터를 확보하기 위해 Docker 이미지 빌드 시간을 측정합니다. 
`time` 명령어를 사용하여 실제 소요 시간을 확인하고, 이후 최적화 결과와 비교할 기준으로 활용합니다.

```bash
time docker build -t petclinic-basic:latest .
```

<br>
<img width="953" height="519" alt="image" src="https://github.com/user-attachments/assets/10c38201-ce6c-4332-8766-2f12078a3c2a" />

<img width="637" height="492" alt="image02" src="https://github.com/user-attachments/assets/e2e0be10-00c5-4902-b142-6a71b1f8d6e7" />



<br>[ 결과 ] **4분 27초**,  **877MB​**

- 이미지의 레이어 구조를 보면, 애플리케이션 코드 자체의 크기는 크지 않지만 **JDK 기반 이미지와 Maven dependency 다운로드 과정**에서 많은 용량이 추가되는 것을 확인할 수 있습니다. <br>
- 특히 **의존성 설치(dependency:resolve) 단계와 JDK 및 시스템 패키지 레이어**가 전체 이미지 크기의 대부분을 차지하고 있으며, 이는 실행에<br> 직접 필요하지 않은 빌드 환경까지 포함되어 있기 때문입니다.

<br>


---


## 3️⃣ 코드 변경 후 재빌드 테스트

Docker의 layer cache 동작을 보다 명확하게 확인하기 위해, 단순한 소스 코드`src` 수정이 아닌 `pom.xml`을 수정한 뒤 다시 빌드를 수행합니다.

일반적으로, `src`만 수정할 경우 **dependency 레이어가 캐시로 유지되기** 때문에 리빌드 속도가 크게 빨라지지만, 이는 Dockerfile이 이미 캐시를 <br>활용하도록 구성되어 있기 때문입니다. 이러한 경우에는 최적화 효과를 명확하게 비교하기 어렵습니다.

이에 따라 의존성 처리 단계부터 캐시가 무효화되도록 **`pom.xml`의 일부를 수정한 후 재빌드를** 진행하여, 캐시 활용 여부에 따른 빌드<br> 성능 차이를 확인합니다.

```bash
# pom.xml 일부 수정 (주석 또는 properties 추가)
nano pom.xml

# 다시 빌드
time docker build -t petclinic-basic:latest .
```

<br>

---


## 4️⃣ 수정 후 리빌드 결과 측정

`pom.xml`을 수정한 후 다시 빌드를 수행하여, 캐시가 무효화된 상황에서의 **빌드 시간과 이미지 상태**를 확인합니다. 이를 통해 dependency 단계까지 다시 수행될 때의 성능 변화를 관찰할 수 있습니다.

```bash
time docker build -t petclinic-basic:latest .
```

<img width="652" height="328" alt="image03" src="https://github.com/user-attachments/assets/c0d060f7-aa2d-4520-85ce-4f6931553a06" />


출력 결과에서 `real` 값은 실제 경과 시간을 의미하며, dependency 단계부터 다시 수행되기 때문에 이전 `src` 수정보다 상대적으로 더 많은 시간이 <br>소요되는 것을 확인할 수 있습니다.

리빌드 결과, **빌드 시간은 증가**하였고 **이미지 크기**는 초기 빌드와 큰 차이를 보이지 않았습니다. 이는 Docker 이미지 크기가 **주로 JDK, 시스템 패키지, dependency 레이어**에 의해 결정되며, 동일한 구조를 유지하는 한 용량 변화는 거의 발생하지 않기 때문입니다.

또한 `docker history`를 통해 확인해보면, **dependency 관련 레이어부터 다시 생성되지만 전체 이미지 구조 자체**는 크게 달라지지 않는 것을<br> 확인할 수 있습니다.

```bash
docker history petclinic-basic:latest
```
<img width="954" height="485" alt="image" src="https://github.com/user-attachments/assets/3a37474c-3da5-4551-9642-a4ca94866570" />

<br>

이러한 결과를 통해 **Docker layer cache**는 파일 변경 위치에 따라 재빌드 범위를 다르게 가져가며, 특히 `pom.xml`과 같이 의존성에 영향을 주는<br> 파일이 변경될 경우 빌드 시간에는 영향을 주지만 이미지 크기에는 큰 변화를 주지 않는다는 것을 확인할 수 있습니다.

---

## 📊 측정 결과

### ⏱️ 빌드 시간

| 구분      | 시간 |
| ------- | -- |
| 기본 빌드   | 4분 27초   |
| 수정 후 빌드 |  4분 29초  |


---
# 경량화 후 실습 과정  

## 1️⃣ 이미지 최적화
- **Multi-stage** 빌드
- **Jlink** 기반 **커스텀 JRE** 생성
- **경량 베이스 이미지** 사용

실행 구조를 단순화하고 불필요한 요소를 제거하여 이미지의 크기 감소와 성능 향상을 목적으로 최적화를 진행하였습니다.

## 최적화 Dockerfile

```dockerfile
FROM eclipse-temurin:21-jdk-jammy AS build

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./

RUN chmod +x mvnw
RUN ./mvnw dependency:go-offline
COPY src ./src

RUN ./mvnw clean package -DskipTests -Dcheckstyle.skip

RUN jar xf target/*.jar && \
    jdeps \
      --ignore-missing-deps \
      --recursive \
      --multi-release 21 \
      --print-module-deps \
      --class-path 'BOOT-INF/lib/*' \
      target/*.jar > deps.info

RUN echo "$(cat deps.info),java.sql,jdk.crypto.ec" > deps.info

RUN jlink \
    --add-modules $(cat deps.info) \
    --strip-debug \
    --compress=2 \
    --no-header-files \
    --no-man-pages \
    --output /custom-jre

FROM debian:bookworm-slim AS runtime

ENV JAVA_HOME=/opt/java/openjdk
ENV PATH="${JAVA_HOME}/bin:${PATH}"

WORKDIR /app

COPY --from=build /custom-jre $JAVA_HOME
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```
## 2️⃣ 최적화 적용
**1. 멀티 스테이지 구성**
- Dockerfile의 구성을 **빌드 단계와 실행 단계**로 나눕니다.
- **빌드 단계**에서 ``eclipse-temurin:21-jdk-jammy`` 기반으로 작성하고 **커스텀 JRE**를 적용했습니다. 
- **실행 단계**에서 ``debian:bookworm-slim``을 사용하며 ``./mvnw clean package``를 통해 실행 가능한 jar 파일 생성합니다.

**2. jlink 기반 커스텀 JRE 생성**
- ``jdeps``로 불필요한 JDK 구성요소를 제거하고 애플리케이션 실행에 필요한 java 모듈만 구성합니다.
- ``jlink``를 통해 **경량 JRE를 생성**해 실행 환경 최소화를 진행합니다.

**3. 경량 베이스 이미지 사용**
- 런타임 단계에서 경량 이미지인``debian:bookworm-slim``사용해 이미지 크기를 감소시킵니다.

**4. 실행 방식 개선**
- 기존: ``spring-boot:run`` 사용 시 컨테이너 내부적으로 빌드 과정이 함께 수행되어 비효율적이었습니다.
- 변경: ``java -jar`` 사용하여 빌드가 끝난 결과물로 서비스 실행합니다.

## 3️⃣ 최적화 Dockerfile 빌드 시간 측정
```bash
docker build -f Dockerfile_new -t dist_image .
```
<img width="838" height="722" alt="image" src="https://github.com/user-attachments/assets/741dc0d9-47c1-4847-9e5d-92bbbdbb29f1" />

<img width="792" height="87" alt="image" src="https://github.com/user-attachments/assets/2857888b-1d4d-4792-add7-c785b3511805" />

<br>[ 결과 ] **7분 3초**,  **116MB​**

- JDK와 Maven 등을 제거하고 실행에 필요한 최소 구성만 포함되어 진행됩니다. <br>
- jlink를 적용하여 실행 전용 이미지 구조로 개선함으로, 실제 배포에 적합한 형태로 최적화합니다.
<br>

## 4️⃣ 기존 이미지와 경량화 이미지 간의 빌드 시간 및 용량 비교
---
기존 이미지와 경량화 이미지를 비교한 결과, 이미지 크기는 **``877MB``** 에서 **``116MB``**로 크게 감소하였습니다.  
빌드 시간의 경우 기존 이미지에 비해 경량화 이미지가 빌드에 있어 상대적으로 더 오래 소요되는 것을 확인할 수 있습니다. <br>
- 이는 경량화 이미지에서 ``jdeps``와 ``jlink``를 활용한 **커스텀 JRE 생성 과정**이 추가되었기 때문이며, 이로 인해 빌드 시간이 증가 되었습니다.

📊 코드 변경 후 측정 결과 

| 구분         | 기본 이미지 | 경량화 이미지         |
| ---------- | ------ | --------------- |
| 코드 수정 전 빌드 | 4분 27초 |  7분    |
| 코드 수정 후 빌드 | 4분 29초 |  6분 50초 |

---

## 📊 Insight

<br>

1. Docker 이미지 최적화를 진행하며, 특히 **이미지 사이즈 감소와 빌드 효율 개선**에 초점을 맞추었습니다. <br>불필요한 레이어를 줄이고, 멀티 스테이지 빌드 및 의존성 처리 구조를 개선하면서 전체적인 이미지 경량화 방향을 확인할 수 있었습니다.

2. 이후에는 Jenkins 등을 활용한 **CI/CD 환경을 연계하여 자동 빌드 및 배포 과정까지 확장**하는 것을 목표로 설정하였습니다. <br> 이를 통해 Docker 기반 빌드 최적화가 실제 서비스 운영 환경과 어떻게 연결되는지까지 고려할 수 있도록 설계할 예정입니다.
<br>

---

## 🛠️ Troubleshooting: `pom.xml` 변경과 빌드 시간

<br>

#### [문제] <br>
`pom.xml`을 수정한 뒤 리빌드를 수행했을 때, 최적화를 적용했기 때문에 빌드 시간이 줄어들 것으로 예상했지만, 실제로 빌드 시간은 유사함을 보였습니다.

#### [해결] <br>
**Docker의 layer cache** 구조 때문으로, pom.xml이 의존성 처리 이전 단계에서 복사되기 때문에 작은 변경만 발생해도 의존성 관련 <br>레이어가 다시 실행되기 때문입니다. <br> <br>
**BuildKit의 cache mount**를 활용하여 .m2 디렉터리를 재사용하면, pom.xml 변경 시에도 모든 의존성을 다시 다운로드하지 않아도 됩니다.<br> 또한 **dependency:go-offline**을 사용하면 플러그인까지 포함한 **의존성을 사전에 확보**할 수 있어 빌드 과정에서의 추가 네트워크 비용을 줄일 수 있습니다. <br><br>
#### <buildkit 사용전><br>
<img width="758" height="602" alt="image" src="https://github.com/user-attachments/assets/2aa29573-184c-47ed-bc53-02af32f6f1b6" /><br>

#### <buildkit 사용후><br>
<img width="754" height="606" alt="image" src="https://github.com/user-attachments/assets/360550a8-7e50-4cee-a280-b520edf13625" />


<br>
결과적으로 pom.xml 변경은 단순 코드 수정보다 빌드 비용에 더 큰 영향을 줄 수 있으며, 이를 고려한 캐시 전략과 <br> Dockerfile 구조 설계가 필요하다는 점을 확인할 수 있었습니다.
<br>
<br>
<br>
<br>

---

