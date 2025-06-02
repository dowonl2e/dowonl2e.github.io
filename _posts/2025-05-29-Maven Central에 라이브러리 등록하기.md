---
title: 내가 만든 모듈을 Maven Central 라이브러리로 등록하기
# author: dowonl2e
date: 2025-05-29 07:00:00 +0800
categories: [Maven Central]
tags: [Maven Central, Gradle, Library]
pin: true
---

## **Maven Central Repository에 가입**

> [https://central.sonatype.com](https://central.sonatype.com){:target="\_blank"}

## **Maven Central 네임스페이스 구성하기**

### 네임스페이스 생성

네임스페이스는 인증 과정이 필요하기 때문에 소유를 증명할 수 있는 도메인에 대해서 입력해야한다. 

하지만, Github를 이용하면 사용 계정의 유무만 확인되면 이용 가능하기에 **`io.github.<사용자명>`**를 입력하면 된다. → 이 후 인증 과정은 거침

![Maven Central Create Namespace]({{site.url}}/assets/img/maven-central-library/1-Create_Maven_Namespace.png)

![Maven Central Namespace]({{site.url}}/assets/img/maven-central-library/2-Maven_Namespace.png)

### **네임스페이스 인증**

Github 기반으로 등록한 네임스페이스 인증은 간단하게 진행할 수 있다.

네임스페이스 등록면 하단에 `Verification Key`{: .text-blue}를 확인할 수 있는데 해당 키 이름으로 Github Repository를 생성한다. 그리고 `Verify Namespace`{: .text-blue} 버튼을 통해 인증 처리를 진행하면 아래와 같이 인증이 완료되는 것을 확인할 수 있다.

![Maven Central Verified Namespace]({{site.url}}/assets/img/maven-central-library/3-Maven_Namespace_Verified.png)

## **GPG (GNU Privacy Guard)를 이용한 키 생성**

Maven Central에 라이브러리를 업로드할 때 파일 위변조, 배포의 신뢰성을 위해 모든 파일에 **`디지털 서명`{: .text-blue}**을 첨부해야 한다.

### **GPG 설치**

OS 버전에 따라 아래 링크를 통해 GPG를 설치할 수 있다. Mac OS의 경우 지원하는 버전이 없으면 **Homebrew**를 이용해 설치할 수 있다.

> [https://gnupg.org/download/index.html#sec-1-2](https://gnupg.org/download/index.html#sec-1-2){:target="\_blank"}

### **GPG 버전 확인**

```bash
$ gpg --version
```

![GPG Version]({{site.url}}/assets/img/maven-central-library/4-GPG_Version.png)

### **GPG Key 생성**

```bash
$ gpg --gen-key
```

![Create GPG Key 1]({{site.url}}/assets/img/maven-central-library/5-Create_GPG_Key.png)

![Create GPG Key 2]({{site.url}}/assets/img/maven-central-library/6-Create_GPG_Key_2.png)

키를 생성하면 아래와 같이 키를 확인할 수 있는데 아래와 같이 키 정보를 확인할 수 있으며, 생성된 키의 뒤 8자리는 KeyID이다. → `키는 이후 Maven Central Repository 배포 과정에 이용`{: .text-orange}

```bash
pub	ed25519 ...
	XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX <- 뒤에서 8자리가 KeyID
uid ...
sub ...
```

### **GPG Key를 공개키 서버에 등록**

```bash
$ gpg --keyserver hkps://keys.openpgp.org --send-keys [[KeyID(공개 키 뒤 8자리)]]
```

![Create GPG Key Push To Server]({{site.url}}/assets/img/maven-central-library/7-GPG_Key_Push.png)

### **signing.pgp 파일 생성**

```bash
$ gpg --export-secret-keys [[KeyID(공개 키 뒤 8자리)]] > [[다운로드 경로]]/signing.pgp
```

signing.pgp 파일의 다운로드 경로는 이후 배포 과정에서 이용한다.

## **Gradle에서의 Maven Plugins**

공식 문서에서 Gradle 이용시 Maven Central에 배포하기 위해 지원하는 플러그인을 확인할 수 있다.

> [https://central.sonatype.org/publish/publish-portal-gradle/#community-plugins](https://central.sonatype.org/publish/publish-portal-gradle/#community-plugins){:target="\_blank"}

### **Gradle 플러그인 설정**

Maven Central 배포를 위해 [`vanniktech/gradle-maven-publish-plugin`](https://github.com/vanniktech/gradle-maven-publish-plugin/) 을 이용하였다.

#### **build.gradle에 플러그인 추가**

플러그인에서 사용하는 Gradle 버전에 따라 달라지므로 아래 사이트를 통해서 Gradle 버전 별 이용 버전을 확인해야한다.

> [https://github.com/vanniktech/gradle-maven-publish-plugin/](https://github.com/vanniktech/gradle-maven-publish-plugin/){:target="\_blank"}

```groovy
plugins {
    id "com.vanniktech.maven.publish" version "0.29.0" // 플러그인 설정
    id 'signing' // GPG 서명 설정
}
```

#### **signing 설정**

Maven으로 배포할 아티팩트에 **GPG 서명**을 추가한다.

```groovy
signing {
    sign publishing.publications
}
```

#### **Maven Central 구성**

```groovy
import com.vanniktech.maven.publish.SonatypeHost

mavenPublishing {
  publishToMavenCentral(SonatypeHost.CENTRAL_PORTAL)
  signAllPublications()
}
```

- **publishToMavenCentral**: 포털을 통한 게시 설정
- **signAllPublications**: 배포하는 전체 출판물에 대해서 디지털 서명 처리

#### **배포 대상 아티팩트(라이브러리) Maven Central Repository 구성**

```groovy
mavenPublishing {
  coordinates([[namspace]], [[artifactId]], [[version]])
}
```

- **namspace**: Maven Central에서 생성한 Namespace 이름
- **artifactId**: 라이브러리 이름
- **versison**: 라이브러리 버전

#### **배포를 위한 POM(Project Object Model) 구성**

POM 구성에 설정할 각 옵션은 주석으로 작성했으며, license의 경우 개인이 자유롭게 사용할 수 있는 라이센스로 **Apache License**로 설정하였다.

```groovy
mavenPublishing {
    pom {
        name = "[[라이브러리 이름]]"
        description = "[[라이브러리 설명]]"
        inceptionYear = "[[시작 연도]]"
        url = "[[프로젝트 사이트 주소(Github 주소)]]"

        licenses {
            license {
                name = "The Apache License, Version 2.0"
                url = "https://www.apache.org/licenses/LICENSE-2.0.txt"
                distribution = "https://www.apache.org/licenses/LICENSE-2.0.txt"
            }
        }

        developers {
            developer {
                id = "[[개발자 ID]]"
                name = "[[개발자 이름]]"
                email = "[[개발자 이메일 주소]]"
            }
            // 개발자 정보를 추가로 기제할 수 있다.
        }

        scm {
            connection = "scm:git:github.com/[[Github 사용자명]]/[[Repository 명]].git" // SCM 읽기 전용 주소 (ex: GitHub HTTPS)
            developerConnection = "scm:git:ssh://github.com:[[Github 사용자명]]/[[Repository 명]].git" // SCM 개발자 전용 주소 (ex: GitHub SSH)
            url = 'https://github.com/[[Github 사용자명]]/[[Repository 명]]' // 프로젝트의 웹 URL
        }
    }
}
```

### **배포 필요 정보**

배포에 필요한 정보를 **`gradle.properties`** 파일에 작성하였다.

Maven Central Username, Password는 Maven Central Repository 사이트에서 사용자 정보를 통해서 토큰  발급 혹은 재발급 받을 수 있으며, 발급시 생성된 토큰의 Username, Password를 입력하면 된다.

**mavenPublishing**에서 **signAllPublications()**를 작성했다면 GPG 정보를 작성해주어야 한다.

```groovy
// Maven Central
mavenCentralUsername=[["Maven Central 토큰의 username"]]
mavenCentralPassword=[["Maven Central 토큰의 password"]]
 
// GPG
signing.keyId=[["공개 키 뒤 8자리 값(KeyID)"]]
signing.password=[["공개 키 발급 시 입력한 GPG Password"]]
signing.secretKeyRingFile=[["signing.pgp 파일을 Export 한 경로"]]/signing.pgp
```

> `이 정보는 외부에 노출되지 않아야 하는 정보로 Github에 올라가지 않도록 .gitignore에 추가해준다.`{: .text-red}

### **플러그인 build.gradle 전체 구성**

```groovy
import com.vanniktech.maven.publish.SonatypeHost

plugins {
    id 'java'
    id "com.vanniktech.maven.publish" version "0.29.0"
    id 'signing'
}

signing {
    sign publishing.publications
}

mavenPublishing {
    signAllPublications()
    publishToMavenCentral(SonatypeHost.CENTRAL_PORTAL)
    coordinates([[namspace]], [[artifactId]], [[version]])

    pom {
        name = "[[라이브러리 이름]]"
        description = "[[라이브러리 설명]]"
        inceptionYear = "[[시작 연도]]"
        url = "[[프로젝트 사이트 주소(Github 주소)]]"

        licenses {
            license {
                name = "The Apache License, Version 2.0"
                url = "https://www.apache.org/licenses/LICENSE-2.0.txt"
                distribution = "https://www.apache.org/licenses/LICENSE-2.0.txt"
            }
        }

        developers {
            developer {
                id = "[[개발자 ID]]"
                name = "[[개발자 이름]]"
                email = "[[개발자 이메일 주소]]"
            }
        }

        scm {
            connection = "scm:git:github.com/[[Github 사용자명]]/[[Repository 명]].git" // SCM 읽기 전용 주소 (ex: GitHub HTTPS)
            developerConnection = "scm:git:ssh://github.com:[[Github 사용자명]]/[[Repository 명]].git" // SCM 개발자 전용 주소 (ex: GitHub SSH)
            url = 'https://github.com/[[Github 사용자명]]/[[Repository 명]]' // 프로젝트의 웹 URL
        }
    }
}

repositories {
    mavenCentral()
}

dependencies {
  ...
}
```

## **Maven Central에 배포하기**

Gradle 전체 설정 이후 아래 명령어를 통해 배포한다. 만약, 인텔리제이를 사용한다면 Gradle 항목에서 `publishing → publishAllPublicationsToMavenCentralRepository`{: .text-blue}을 실행한다.

```bash
./gradlew clean build publishAllPublicationsToMavenCentralRepository
```

배포가 정상적으로 이루어지면 Deploments 화면에 배포된 정보들이 나오게 된다. 배포만 이루어졌으며 우측의 Publish 버튼을 선택하여 라이브러리를 Publish 한다.

![Maven Central Repository Publish]({{site.url}}/assets/img/maven-central-library/8-Maven_Central_Repository_Publish.png)

## **Maven Central에 출판된 라이브러리 확인하기**

이전에 개발한 엑셀 다운로드 모듈을 Maven Central Repository에 올려보았다. 정상적으로 라이브러리가 출판된 이후 별도 프로젝트를 생성하여 의존성 추가를 하면 아래와 같이 라이브러리가 등록된 것을 확인할 수 있다.

![Add Library Dependency Test]({{site.url}}/assets/img/maven-central-library/9-Library_Dependency_Test.png)
