---
layout: post
title: "나무위키 덤프파일 사용법"
date: 2015-11-05 17:53:42
---

이 문서에서는 [나무위키](https://namu.wiki)에서 주기적으로 공개하는 [데이터베이스 덤프 파일](https://namu.wiki/w/%EB%82%98%EB%AC%B4%EC%9C%84%ED%82%A4:%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4%20%EB%8D%A4%ED%94%84)을 mysql로 import하는 방법을 설명합니다. 기준 환경은 다음과 같습니다:

- os: ubuntu linux 15.04 (64bit)

    위는 제가 사용하는 환경일 뿐이고, 대체로 **mysql이 설치된 linux면 아무 문제가 없**을 겁니다.

- mysql: mysql-server 5.6.27

    역시 제가 사용하는 환경이고, 극단적으로 오래된 버전이 아니라면 아무런 문제가 없을 겁니다.

아래에서 사용되는 변수의 의미는 아래와 같습니다. 콘솔에 입력할 때 ${변수명} 꼴로 표시된 부분을 각자가 사용하시는 값으로 치환해 주시면 됩니다.

- root-username: mysql 관리자 계정 이름. 보통 root.
- root-password: mysql 관리자 계정 비밀번호.
- namu-db: 나무위키 데이터베이스 이름. (필자는 'namu' 사용)
- namu-username: 나무위키 데이터베이스 관리계정 이름. (필자는 'namu' 사용)
- namu-password: 나무위키 데이터베이스 관리계정 비밀번호.

### 데이터베이스 생성

나무위키 대소문자를 구분하는 데이터베이스 설정을 사용하며, 덤프 파일 인코딩은 utf-8 입니다. 따라서 데이터베이스 생성시 character set과 collate를 별도 지정해 줘야 합니다.

우선 root 권한을 가진 계정으로 mysql에 접속합니다:

```sh
mysql -u${root-username} -p${root-password}
```

${namu-db}이름의 데이터베이스를 생성합니다:

```sql
CREATE DATABASE IF NOT EXISTS ${namu-db} DEFAULT CHARACTER SET = 'utf8' DEFAULT COLLATE 'utf8_bin';
```

### 사용자 생성 및 권한 부여

```sql
CREATE USER ${namu-username}@localhost IDENTIFIED BY '${namu-password}';
GRANT ALL PRIVILEGES ON namu.* TO ${namu-username}@localhost;
FLUSH PRIVILEGES;
EXIT;
```

### 덤프 파일 수정

나무위키 덤프 파일의 상단 schema 정의는 아래와 같습니다:

```sql
CREATE TABLE `documents` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `document` varchar(190) NOT NULL,
  `rev` int(11) NOT NULL,
  `text` NOT NULL,
  `date` int(11) NOT NULL,
  PRIMARY KEY (`id`),
);
```

잘 보면 아시겠지만 이 부분에 아래와 같은 문제가 있기 때문에, import 하기 전에 약간 수정을 해 줘야 합니다. (바로 import를 시도하면 아름다운 에러 메시지를 확인하시게 될 겁니다.)

1. text 칼럼의 데이터 형식이 정의되어 있지 않음: TEXT 형식임을 지정해 줘야 합니다. MEDIUMTEXT나 LONGTEXT도 가능하겠지만 딱히 권장하지 않습니다.
2. PRIMARY KEY (`id`) 뒤에 있는 comma가 삭제되어야 합니다.
3. (옵션) index가 정의되어 있지 않음: 문서 제목이 저장되는 document 칼럼에 index를 걸어 주는 것이 여러 모로 편리합니다.

~~도대체 어떻게 덤프를 뜨면 이렇게 나오는지 궁금하다.~~ ~~운영자님 내게 얘기해 봐요. 대체 왜 이랬어요?~~

아래와 같이 압축을 푼 dump 파일에 sed 치환 명령을 주면 위 문제들이 수정된 namu.sql 파일을 얻을 수 있습니다:

```sh
sed 's/`text` NOT NULL,'/'`text` TEXT NOT NULL,/g; s/PRIMARY KEY (`id`),'/'PRIMARY KEY (`id`),\n  INDEX `documents_document` (`document`)/g' namuwiki_20xxxxxxxxxxxx.sql > namu.sql
```

### import

이제 남은 것은 수정된 sql 파일을 위에서 생성해 준 데이터베이스에 import 해주는 것 뿐입니다:

```sh
mysql -u${namu-username} -p${namu-password} --database=${namu-db} < namu.sql
```

### 테스트

이제 제대로 import 되었는지 확인해 보도록 하겠습니다. mysql에 접속합니다:

```sh
mysql -u${namu-username} -p${namu-password} --database=${namu-db}
```

아래 명령어로 [iOS]() 문서의 내용을 확인하실 수 있습니다.

```sql
SELECT * FROM documents where document = 'iOS';
```

아래 명령을 내리면 ios 항목이 iOS 항목으로 리다이렉트[^1] 되는 표제어임을 확인할 수 있습니다. 위에서 데이터베이스가 대소문자를 구분하도록 collate값을 설정했기 때문에 iOS와 다른 결과가 나오는 것이 맞습니다.

```sql
SELECT * FROM documents where document = 'ios';
```

마찬가지 이유에서 아래와 같은 명령을 내리면 결과가 나오지 않습니다.

```sql
SELECT * FROM documents where document = 'iOs';
```

[^1]: #redirect iOS
