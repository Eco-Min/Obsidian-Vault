 Docker 를 이용하여 설치할 예정

- Docker Registry 에서 Redis 이미지 불러오기
> docker pull redis

- Docker Redis 실행
> docker run -p 6379:6379 redis
> docker run --name my-redis -d -p 6379:6379 redis

- Docker Redis stop / start
>docker stop my-redis
>docker stop my-redis

- Docker Redis remove
> docker rm my-redis

## Redis 모듈
- redis-server : 레디스 서버
- redis-cli : 레디스 서버에 커맨드를 실행할 수 있는 인터페이스
-> container 안에서 redis-clifmf 실행 시켜야 한다.

- Docker Container shell 실행
> docker exec -it my-redis /bin/sh

- Container 내부 쉘에서 Redis-cli 실행
 > redis-cli (host, port 지정 하지 않으면 127.0.0.1:6379 사용)
 > reids-cli -h xxxx -p xxxx