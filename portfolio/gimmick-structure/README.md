# 기믹 공통 구조와 플레이어 적용 정책

**담당자:** 최성원 — 공통 구조 설계 / 팀원 — 기믹별 세부 동작 구현

## 코드

**ItemBehavior:** 색상 정책과 플레이어 진입 이벤트를 공통 처리합니다. **PlayerSelectableInterface:** 색상별 적용 정책을 교체합니다. **Consumable:** 공통 소모 처리를 확장합니다. 개별 기믹의 효과는 이 샘플에 포함하지 않습니다.

```csharp
// 색상 판정을 기믹 효과와 분리하는 공통 계약
public interface PlayerSelectableInterface
{
    bool CheckColor(Collider other, int playerId);
}

// 공용 기믹: 모든 플레이어 허용
public class PlayerNonSelect : PlayerSelectableInterface
{
    public bool CheckColor(Collider other, int playerId) => true;
}

// 전용 기믹: 핑크/블루 플레이어만 허용
public class PlayerSelectable : PlayerSelectableInterface
{
    public bool CheckColor(Collider other, int playerId)
        => playerId == 1 && other.CompareTag("Player1")
        || playerId == 2 && other.CompareTag("Player2");
}
```

```csharp
// ItemBehavior: 원본의 색상별 외형 설정은 생략
protected virtual void Awake()
{
    playerSelectable = color == PlayerColor.noneSelectable
        ? new PlayerNonSelect() : new PlayerSelectable();
}

// 접촉 → 플레이어/색상 확인 → 구독한 효과 전달의 공통 흐름
protected virtual void OnTriggerEnter(Collider other)
{
    if (!other.CompareTag("Player1") && !other.CompareTag("Player2")) return;
    if (!playerSelectable.CheckColor(other, (int)color)) return;
    // 공통부는 적용 가능 여부만 결정하고 실제 효과는 구독한 하위 기믹에 맡긴다.
    PlayerTriggerEvent?.Invoke(other);
}

// Consumable: 공통 흐름을 실행하고 같은 적용 조건으로 소모 처리
protected override void OnTriggerEnter(Collider other)
{
    base.OnTriggerEnter(other);
    // base의 void 호출만으로 성공 여부를 알 수 없어 소모 조건도 다시 확인한다.
    if (!other.CompareTag("Player1") && !other.CompareTag("Player2")) return;
    if (!playerSelectable.CheckColor(other, (int)color)) return;
    Destroy(gameObject);
}
```

## 생략 및 재구성한 내용

- 원본: `Assets/Scripts/Items/ItemBehavior.cs`, `PlayerSelectableInterface.cs`, `PlayerSelectable.cs`, `PlayerNonSelect.cs`, `Consumable.cs`. 기준 커밋: `0607414498b6af712b2c6225317c23dc2fa48ac7`.
- 이 항목은 공통 구조 설계 범위를 보여줍니다. Frog·GPause·Pause·Portal·SpringScript·Swap 등의 세부 효과는 개인 구현으로 제시하지 않습니다.
- Awake의 색상별 Animator·Sprite 설정을 생략하고, 원본의 타입 검사 후 CheckColor를 항상 호출하는 형태로 정리했습니다. 
- 하위 기믹은 원본에서 base.Awake 후 PlayerTriggerEvent에 자신의 효과를 등록합니다. 세부 효과와 리소스는 제외했습니다.
- 두 번째 블록의 ItemBehavior 메서드와 Consumable 메서드는 서로 다른 클래스의 발췌입니다. PlayerColor·이벤트·필드·Unity Collider·태그는 원본 의존성으로 단독 실행되지 않습니다.
