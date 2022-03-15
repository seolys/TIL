### 22.3.14
- 다른 것은 틀린게 아니다.
- 고칠 수 있는 버그는 다시 재현할 수 있는 버그.
- 훌륭한 프로그래머는 인간이 이해하는 코드를 작성한다.

### 22.3.15
- tryCatch에서 Throwable로 catch하지말자. OutOfMemory 등 JVM예외까지 catch된다.
- Exception으로 catch하지말고 최대한, 그리고 가장 구체적인 예외를 잡자.(상위타입으로 잡지 말자)
  - NullPointException같은 잡고 싶지않은(잡히지 않아야할) Exception까지 같이 잡힌다. 
- 원인사슬을 깨지말자. 다른 Exception으로 던질경우, 생성자에 원인이 된 Exception를 꼭 넘기자.
  - Stacktrace를 유지시켜야 한다.
- 응답값에 Optional을 적극 활용하자.
  - 값이 무조건이 있는경우에는 Optional이 필요없다.
  - Optional은 값이 있을수도 있고 없을 수도 있다는 의미를 전달한다.
- 파라미터에 Optional을 넘기지 말자.
  - Optional이 아닐경우, 상태가 2개: 널 or 값
  - Optional일 경우, 상태가 3개: Optional이 null or 빈 Optional or 값이 있는 Optional
