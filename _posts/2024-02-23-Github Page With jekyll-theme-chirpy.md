---
title: Github Page With jekyll-theme-chirpy
# author: dowonl2e
date: 2024-02-23 12:00:00 +0800
categories: [Jekyll]
tags: [Github Page]
pin: true
img_path: "/posts/20240223"
---

## **Github Page Repository 생성**

아래 링크에서 파일을 다운로드 받거나 Web URL을 복제하여 로컬 리포지토리를 만들어준다. fork 후 Repository를 Clone해도 됩니다.

- Theme : http://jekyllthemes.org/themes/jekyll-theme-chirpy
- Github : https://github.com/cotes2020/jekyll-theme-chirpy?tab=readme-ov-file

Github Page 생성시에는 Repository명은 아래와 같이 설정해줍니다.

- \<username\>.github.io

> 구버전에는 Github Actions에 작업이 완료되면 gh-pages Branch가 자동 생성되었지만, main으로도 가능합니다.
> {: .prompt-tip }

## **설정 및 구성 확인**

### **Repository Clone**

1. 디렉토리로 이동 후 Git 연동을 해줍니다. (계정 및 기타 설정은 생략)

```bash
$ git clone <Github Page Clone Web URL>
```

2. 복제한 파일 안에 보면 tools 폴더가 있고, init 파일을 명령어 또는 실행
3. 저는 VSCode를 사용했지만, Subline Text([https://www.sublimetext.com](https://www.sublimetext.com))과 같은 프로그램을 사용하셔도 됩니다.
   ![프로젝트 구조 확인]({{site.url}}/assets/img/jekyll_chirpy_github_page/1_project open.png)

### **\_config.yml 확인**

- lang : 언어
- timezone : 타임존
- title : 사이드바 타이틀
- tagline : 사이드바 설명
- url : 깃허브 페이지 URL
- github.username : 깃허브 계정명
- social.name : 이름
- social.email : 이메일
- social.links : 깃허브, 트위터 등
  - 이 링크 표시는 사이드바 하단 아이콘(Github, Twitter 등) 연동
- img_cdn : 프로필 이미지 주소
- avatar : 프로필 경로 및 파일

### **게시물 및 네이밍 규칙**

- 경로 : **\_posts** 디렉토리 안에 생성
- 파일 네이밍 규칙 : **yyyy-MM-dd-{제목}.md**
- 게시물 정렬은 샘플 파일 안에 date 기준으로 내림차순

## **min.js 파일 생성**

실행하게 되면 정상적으로 화면이 출력되지만 검색, 사이드바 리스너 등 일부 기능이 동작하지 않는다. 브라우저 콘솔 로그를 보면 home.min.js 파일을 찾을 수 없다는 오류도 확인된다. node, npm 설치 후 min.js 파일을 생성해준다.

### **node 설치가 되어있지 않다면 아래 명령어를 통해 설치**

```bash
$ brew install node
```

![Node & NPM 버전 확인]({{site.url}}/assets/img/jekyll_chirpy_github_page/2_node-npm version.png)

### **프로젝트 루트 경로에서 아래 명령어 실행하여 min.js 파일들을 생성**

```bash
$ NODE_ENV=production npx rollup -c --bundleConfigAsCjs
```

![min js 파일 생성]({{site.url}}/assets/img/jekyll_chirpy_github_page/3_create minjs.png)

### **Commit & Push전에 .gitignore 파일에 아래 파일 Misc 부분 주석**

```
# Misc
# assets/js/dist # => 주석
```

## **Github Page 적용**

### **Commit & Push**

```bash
$ git add -A
$ git commit -m “{커밋 메세지}”
$ git push
```

### **Github Actions**

![첫 Github Actions 실행]({{site.url}}/assets/img/jekyll_chirpy_github_page/4_First Github Action.png)

### **Repository Settings 설정**

Github Action이 완료되더라도 index.html의 텍스트가 그대로 출력된다. 리포지토리에 **Settings > Code and automation > Pages**로 이동 후 **Build and deployment**에 **Source**를 **Guthub Actions**으로 변경해줍니다.

![Repository 설정]({{site.url}}/assets/img/jekyll_chirpy_github_page/5_Repository Settings.png)

바로 아래에서 Jekyll.yml 파일을 설정하면 .github > workflows에 파일이 생성되며 Github Actions에서 작업이 진행된다.

![Jekyll Github Actions 실행]({{site.url}}/assets/img/jekyll_chirpy_github_page/6_Jekyll Actions.png)

Jekyll.yml 작업이 끝나고 https://\<username\>.github.io에 접속하면 아래와 같이 사이트가 정상적으로 출력되는 것을 확인하실 수 있습니다. 수정한 내용이 있을 경우 Git에 Push하면 자동으로 Github Actions에 작업이 실행되고 작업이 완료되면 수정한 내용을 확인할 수 있다.

![Github Page]({{site.url}}/assets/img/jekyll_chirpy_github_page/7_github page.png)
