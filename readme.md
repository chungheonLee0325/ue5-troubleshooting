# UE5 Troubleshooting

언리얼 엔진 5 개발 중 마주친 문제들의 원인 분석과 해결 과정.
엔진 소스코드 기반으로 근본 원인을 파악합니다.

## Articles

> ### [AnimNotifyState 인스턴스 공유 문제](./animnotifystate-instance-sharing.md)  
> 여러 캐릭터가 같은 NotifyState 인스턴스를 공유해서 생기는 문제와 해결

> ### [Fast Array로 인벤토리 동기화하기](./fast-array-inventory.md)  
> TMap 리플리케이션 불가 문제, FFastArraySerializer로 델타 동기화 구현

> ### [Listen Server에서 소켓 기반 공격이 실패하는 이유](./listen-server-lod-socket.md)  
> LOD가 소켓 위치 계산에 미치는 영향과 SetForcedLOD 해결책

> ### [BindWidget vs GetWidgetFromName](./bindwidget-vs-getwidgetfromname.md)  
> GetWidgetFromName의 O(N) 트리 순회 비용, 거의 안 쓰는 위젯도 BindWidget이 효율적인 이유

> ### [UScrollBox 안에 버튼을 배치했을 때 드래그 스크롤이 안 되는 이유](./scrollbox-button-drag.md)  
> Slate 이벤트 버블링과 FReply, PreciseClick이 드래그 스크롤을 가능하게 하는 원리
> > #### [터치와 마우스 입력 처리 차이](./scrollbox-button-drag/touch-vs-mouse.md)  
> > OnPreviewMouseButtonDown(Tunnel)과 OnMouseButtonDown(Bubble)의 차이, 터치 전용 변수들
