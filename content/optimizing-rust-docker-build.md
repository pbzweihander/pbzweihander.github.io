+++
title = "Rust Docker Build 시간 및 크기 압축하기"
date = 2019-01-16
+++

> Reference: http://whitfin.io/speeding-up-rust-docker-builds/

Rust로 된 애플리케이션을 Docker Image로 감쌀 때, Rust 빌드의 특성상 빌드 시간이 굉장히 길어지는 문제가 있는데, 이를 줄이고 또한 musl을 통해 이미지 크기 까지 줄이는 법을 알아봅시다.

먼저 프로젝트 이름은 `project_name`으로 가정합니다. 이 부분을 각자의 프로젝트 명으로 바꿔주세요.

## Base Dockerfile

기본적으로 Dockerfile을 작성한다면 다음과 같습니다.

```Dockerfile
FROM rust:1.31

COPY ./Cargo.toml ./Cargo.toml
COPY ./Cargo.lock ./Cargo.lock
COPY ./src ./src

RUN cargo build --release

CMD [ ".target/release/project_name" ]
```

그런데 이렇게 하면 매 docker build마다 dependency를 또 다시 받고 처음부터 빌드하면서 시간이 굉장히 오래 걸리게 됩니다. 이를 줄여봅시다.

## Optimizing Build Times

Docker에는 docker layer라는 좋은 기능이 있어서, Dockerfile의 한 줄의 실행 결과가 저장됩니다.
그래서 만약 우리가 dependency를 빌드하는 과정을 docker layer로 저장할 수 있다면, 빌드 시간이 빨라지겠죠?

```Dockerfile
FROM rust:1.31

# 먼저 빈 프로젝트를 생성합니다
RUN USER=root cargo new --bin project_name
WORKDIR /project_name

# manifest 파일들을 복사합니다
COPY ./Cargo.lock ./Cargo.lock
COPY ./Cargo.toml ./Cargo.toml

# 여기서 빈 main.rs 파일을 이용해 한번 빌드함으로써,
# dependency를 다운받고 한번 빌드하는 과정이 docker layer 상에 저장됩니다
RUN cargo build --release
RUN rm src/*.rs

# 실제 코드들을 복사합니다
COPY ./src ./src

# 임시로 빌드했던 파일을 삭제하고 다시 빌드합니다
RUN rm ./target/release/deps/project_name*
RUN cargo build --release

CMD ["./target/release/project_name"]
```

위 방법대로, 한번 빈 `main.rs` 파일 (실제로는 hello world 코드가 들어있겠죠)로 한번 빌드하면,
dependency들을 전부 다운로드하고 컴파일하게 됩니다.
만약 코드만 수정하고 다시 docker image를 빌드한다면 dependency를 다시 빌드하는 과정은 docker layer에 의해
생략되고 바로 수정된 코드만 빌드하게 됩니다.

## Optimizing Build Sizes

### Without musl

하지만 이 상태로 docker image를 그대로 배포하기에는 build artifact가 너무 많아 image 사이즈가 너무 커지게 됩니다.
Docker에선 이것을 해결하기 위해 multi-stage build를 가능하게 하는데요.

```Dockerfile
FROM rust:1.31

RUN USER=root cargo new --bin project_name
WORKDIR /project_name

COPY ./Cargo.lock ./Cargo.lock
COPY ./Cargo.toml ./Cargo.toml

RUN cargo build --release
RUN rm src/*.rs

COPY ./src ./src

RUN rm ./target/release/deps/project_name*
RUN cargo build --release

# 아무것도 없는 새 rust 이미지에서 시작합니다
FROM rust:1.31

# 이전 stage에서 빌드 결과만을 가져옵니다
COPY --from=0 /project_name/target/release/project_name .

CMD ["./project_name"]
```

이렇게 하면 마지막에 바이너리만을 가지고 이미지를 새로 만들었으니 마지막 이미지의 크기가 아주 작아지게 됩니다.
하지만 여기서 끝이 아닙니다. 왜 아무것도 없는 바이너리만을 들고 있을 건데 base image가 rust여야 할까요?
debian이면 더 좋겠죠? 근데 만약 더 줄이고 싶다면?

### With musl

Docker에는 `scratch` 라는 이름의, 정말 아무것도 없는 base image가 있습니다.
근데 여기는 정말 아무런 library도 없기 때문에, 우리 바이너리가 필요한 library들을 직접 설치해줘야 합니다.

아니면.. musl을 통해 모든 dependency를 한 binary 안에 통합시켜버리는 방법이 있죠.

```Dockerfile
# muslrust를 통해 빌드합니다
FROM clux/muslrust:1.31.0-stable

WORKDIR /
RUN USER=root cargo new --bin project_name
WORKDIR /project_name

COPY ./Cargo.lock ./Cargo.lock
COPY ./Cargo.toml ./Cargo.toml

RUN cargo build --release
RUN rm src/*.rs

COPY ./src ./src

# 여기서 target 폴더가 조금 바뀌게 됩니다
RUN rm ./target/x86_64-unknown-linux-musl/release/deps/project_name*
RUN cargo build --release

# 이제 scratch를 base image로 이용할 수 있습니다
FROM scratch

COPY --from=0 /project_name/target/x86_64-unknown-linux-musl/release/project_name .

CMD [ "./project_name" ]
```

[clux/muslrust](https://github.com/clux/muslrust)를 이용해 static binary를 만들 수 있습니다.
static binary가 무엇인지 musl이 무엇인지는 여기서 자세히 다루지는 않겠습니다. (저도 잘 모르기 때문에..)
