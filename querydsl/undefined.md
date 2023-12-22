# 셋팅

query DSL 를 사용하기 위해선 셋팅을 해야하는데,

처음에 할땐 조금 해맬수도 있다.



## 초기 셋팅



## 프로젝트를 내려받았을때

<figure><img src="../.gitbook/assets/스크린샷 2023-12-22 오후 6.40.39.png" alt=""><figcaption></figcaption></figure>

프로젝트를 내려받게되고 build.gradle 를 해도 위의 Q 클래스는 적용되지 않는다.

<figure><img src="../.gitbook/assets/스크린샷 2023-12-22 오후 7.59.44.png" alt=""><figcaption></figcaption></figure>

```markdown
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.0.1'
    id 'io.spring.dependency-management' version '1.1.0'
    // querydsl관련 명령어를 gradle탭에 생성해준다. (권장사항)
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

jar {
    enabled = false
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
    maven { url 'https://repo.spring.io/snapshot' }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'mysql:mysql-connector-java:8.0.28'
    implementation 'org.springframework.boot:spring-boot-devtools'
    implementation 'org.hibernate.validator:hibernate-validator'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.0.2'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'	//for jpa
    implementation 'io.jsonwebtoken:jjwt-impl:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-jackson:0.11.5'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    // test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'com.h2database:h2'

    // QueryDSL Implementation
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}

test {
    useJUnitPlatform()
}

/**
 * QueryDSL Build Options
 */
def querydslDir = "src/main/generated"

sourceSets {
    main.java.srcDirs += [ querydslDir ]
}


tasks.withType(JavaCompile) {
    options.getGeneratedSourceOutputDirectory().set(file(querydslDir))
}

clean.doLast {
    file(querydslDir).deleteDir()
}

tasks.named('test') {
    useJUnitPlatform()
}
```

