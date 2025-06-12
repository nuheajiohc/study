# SpringBatch ItemReader 계층 구조

Spring Batch에서 하나의 Step은 크게 두 가지 처리 방식으로 나뉜다.  

- `Tasklet` 기반 처리
- `Chunk-oriented Processing` 기반 처리

`Chunk-oriented Processing` 기반 처리 방식도 내부적으로는 `Tasklet` 인터페이스를 구현한 방식이지만 편의상 분리해두었다.  
두 가지 방식 중 널리 쓰이는 방식은 `Chunk-oriented Processing` 기반 처리 방식이다.

chunk 방식은 크게 `ItemReader -> ItemProcessor(선택) -> ItemWriter` 구조로 이루어지며, 이 글에서는 `ItemReader`의 계층 구조에 대해서 알아보겠다.

