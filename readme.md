# UE5 Troubleshooting

언리얼 엔진 5 개발 중 마주친 문제들의 원인 분석과 해결 과정.
엔진 소스코드 기반으로 근본 원인을 파악합니다.

## Articles

> ### [AnimNotifyState 인스턴스 공유 문제](./animnotifystate-instance-sharing.md)  
> 여러 캐릭터가 같은 NotifyState 인스턴스를 공유해서 생기는 문제와 해결

> ### [Fast Array로 인벤토리 동기화하기](./fast-array-inventory.md)  
> TMap 리플리케이션 불가 문제, FFastArraySerializer로 델타 동기화 구현

> ### [Listen Server에서 Client가 소켓 기반 공격이 실패하는 이유](./listen-server-lod-socket.md)  
> LOD가 소켓 위치 계산에 미치는 영향과 SetForcedLOD 해결책
