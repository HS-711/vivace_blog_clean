---
title: "ShellShock"
description: "Shell-shock"
date: 2026-06-29
update: 2026-05-01
tags:
  - InfoSec
  - Web Security

series: "Web Security"
---

Shellshock은 대표적인 Command Injection 공격이다.<br/>
CVE-2014-6271로 알려져있다.<br/>

환경변수에 대한 함수정의 이후 임의의 코드가 실행되었을 때, 보안 취약점을 드러나게 하는 공격이다.<br/>
Apache CGI환경에서 불특정 사용자가 위의 임의의 코드를 실행하였을때, 특수 권한 등이 침해당할 수 있다. <br/>

>CGI?
웹을 단지 static하게 보여주는 것이 아니라, 웹 브라우저의 요청을 받아<br/>
서버상의 프로그램(shellscript, ...) 등을 실행하여 웹을 dynamic하게 보여주는 protocol.<br/>
환경변수 형태로 요청을 처리하여 동적으로 반영할 수 있다는 장점이 있지만,<br/>
스크립트가 내부적으로 bash를 실행한다는 취약점이 있다. <br/>

```bash
env x='() { :;}; echo vulnerable' bash -c "echo this is a test" 
> vulnerable
```

원래의 의도는 환경 변수 x에 '() { :;}; echo vulnerable'를 저장하고,<br/>
새로운 bash를 실행하여 코드가 실행되는 것이다.


그러나 취약점이 있는 bash는 { :;};까지만 보고 bash는 함수를 정의하게 된다.<br/> 
또한, 이로 인해 그 뒤를 즉시 실행해야 하는 명령어로 오인한다.<br/>
따라서 코드의 의도와 대조되어 'vulnerable'을 출력한다.


**References**<br/>
CVE-2014-6271, https://www.cve.org/CVERecord?id=CVE-2014-6271