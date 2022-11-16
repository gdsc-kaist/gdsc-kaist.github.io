---
layout: single
title:  Welcome to GDSC KAIST
date:   2022-10-30 18:30:26 +0900
author: Bongjun Jang

header:
  og_image: /assets/images/logo.jpg
---

# Overview

RustPython Contribution 프로젝트 첫번째 미팅에서 공유한 내용입니다.
Python의 인터프리터를 개괄적으로 이해하고, 인터프리터에서 입력 문자열을 분석하는 첫 단계인
Lexical Analysis에 대해서 내용을 공유하였고, 이와 관련해 가장 대중적인 문자열 인코딩인 UTF-8에 대해서 이야기했습니다.

## Intrepreted Language vs. Compiled Language

프로그래밍 언어는 실행되는 방식에 따라 인터프리터 언어와 컴파일러 언어로 나눌 수 있다.
컴파일러 언어는 C, C++, Java와 같이 실행되기 전 어셈블리로 변환되거나, 가상머신에 동작하기 위한 IR(Intermidate Representation)으로 변환되는 과정을 겪은 후,
운영체제에서 프로세스로 실행되거나, JVM과 같은 가상머신에서 실행된다.
인터프리터 언어는 이와 달리 프로그램을 한줄씩 바로 실행하는 형태를 띈다.
Scala와 같은 언어는 컴파일도 가능하지만 REPL을 통해 한줄씩 실행하는 것도 가능하므로 구분에 유의하여야 한다.

## Intrepreter의 구조

이 프로젝트에서는 Python 인터프리터에 집중하기로 하였다.
우선 Python 프로그램은 실행되면 다음과 같은 과정을 겪는다.

1. Lexer (Tokenizer)에 의해 프로그램이 a Stream of Tokens로 변환된다.
2. Parser에 의해 토큰들이 AST로 변환되고, 바이트코드가 생성된다.
3. VM에 의해 바이트코드가 실행된다.

## Lexer

이번 미팅에서는 Lexer에 대해 이해하기로 하였다.
Lexer의 동작을 이해하기 위해서는 우선 문자열에 대한 이해가 필요하다.

