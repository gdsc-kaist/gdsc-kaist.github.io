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

## UTF-8

이번 미팅에서는 Lexer에 대해 이해하기로 하였다.
Lexer의 동작을 이해하기 위해서는 우선 문자열에 대한 이해가 필요하다.
Python의 기본 인코딩 방식이 UTF-8이기 때문에 UTF-8 문자열이 무엇인지 살펴보자.

UTF-8는 유니코드의 한 인코딩 방식인데 가장 기초적인 문자열 구성방식인 ASCII와 다르게 한 문자는 여러 바이트로 구성될 수 있다.
예를 들어 일반적으로 '안' 이라는 문자는 3바이트를 사용해 인코딩한다.


| First code point | Last code point |  Byte 1  |  Byte 2  |  Byte 3  |  Byte 4  |
| :--------------: | :-------------: | :------: | :------: | :------: | :------: |
|      U+0000      |     U+007F      | 0xxxxxxx |
|      U+0080      |     U+07FF      | 110xxxxx | 10xxxxxx |
|      U+0800      |     U+FFFF      | 1110xxxx | 10xxxxxx | 10xxxxxx |
|     U+10000      |    U+10FFFF     | 11110xxx | 10xxxxxx | 10xxxxxx | 10xxxxxx |

Reference: https://en.wikipedia.org/wiki/UTF-8

하나의 바이트만을 사용할 경우 ASCII와 똑같은 인코딩을 가지기 때문에 ASCII 문자열은 UTF-8을 이용해 바로 인코딩될 수 있다.

주목할 만한 점은 한 문자가 얼마나 많은 바이트를 사용하는지를 알기 위해 문자의 첫번째 바이트를 살펴보아야 한다는 뜻이다.
ASCII에서는 모든 문자가 1바이트로 인코딩되기 때문에 Constant Time Encoding이 가능했다. (세번째 문자를 가져오기 위해 `string[2]`이라고 하면 된다.)
하지만 UTF-8에서는 한 문자가 얼마나 많은 바이트를 사용하는지 문자열을 읽기 전에 알 수 없기 때문에 Linear Search가 불가피하다.

이러한 UTF-8의 특성은 Rust의 `String` 타입에 구현되어 있다.
Rust의 `String`이나 `&str`에 `string[2]`와 같이 인덱싱을 하려고 하면 컴파일 에러가 나온다.
3번째 문자를 얻으려하는 것인지, 3번째 바이트를 얻으려하는 것인지 그 의미가 모호하기 때문이다.
따라서 러스트에서는 문자열 처리를 위해 해당 타입에 `Iterator`를 구현해 `next(&self mut) -> Option<char>` 메소드를 통해 다음 문자를 얻을 수 있도록 하고 있다.

```rust
let s = String::from("안녕하세요");
s[2]; // Compile Error! `String` is not indexable

let mut chars = s.chars(); // Returns an iterator which produces `char` every `next()` calls
let ahn = chars.next().unwrap(); // '안'
let nyeong = chars.next().unwrap(); // '녕'
```

또 하나 짚고넘어가야할 부분은 `String`과`&str`에서 인덱싱을 불가해도 슬라이싱은 가능하다는 점이다.
슬라이싱은 연속적인 바이트 구조에 대해 바이트의 시작과 끝을 지정해 해당 범위에 대한 포인터를 반환하는 기능이다.
하지만 슬라이싱에 경우에도 바이트의 시작과 끝이 유효한 UTF-8이 아니라면 런타임에러를 발생시킨다.
슬라이싱을 통해 얻은 데이터의 타입이 `&str`인데, Rust에서는 `str`이 유효한 UTF-8 문자열임을 보증하기 때문에 이와 같은 에러가 발생한다.

```rust
let s = String::from("안녕하세요");
let wrong_ahn = s[0..2]; // Run-time error! '안' takes 3 bytes, but 2 bytes specified in the slice
let good_ahn = s[0..3]; // '안', and the type is `&str`
```


## RustPython의 Lexer 구현과 CPython의 구현

