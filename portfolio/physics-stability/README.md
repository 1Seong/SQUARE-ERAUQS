# 빙판 벽 연결과 착지 판정

**담당자:** 최성원 — 연결 상태 전파 및 스프링 미발동 원인 수정

## 코드

**WallConnection:** 좌우 이웃이 벽과 연결됐는지 확인하고 스프링의 X축 이동을 제한합니다. **PlayerJump:** 접촉면 normal로 실제 바닥 착지만 인정합니다. **Player · SpringMove · Movable:** 빙판 진입·이탈과 이동을 처리합니다.

```csharp
// WallConnection.Update: 시점에 맞춘 BoxCast로 양쪽 연결 상태 갱신
private void Update()
{
    if (!_doUpdate) return;
    var halfExtents = GameManager.instance.isSideView
        ? new Vector3(0.01f, 0.41f, 7.5f) : new Vector3(0.01f, 0.1f, 0.41f);
    int mask = LayerMask.GetMask("Platform");
    var left = FindConnection(Physics.BoxCastAll(
        _rigid.position + Vector3.left * 0.47f, halfExtents,
        Vector3.left, Quaternion.identity, 0.5f, mask));
    var right = FindConnection(Physics.BoxCastAll(
        _rigid.position + Vector3.right * 0.47f, halfExtents,
        Vector3.right, Quaternion.identity, 0.5f, mask));
    _leftColl = left;
    _rightColl = right;
    connectedToWall = left != null || right != null;
}

// 벽 자체 또는 벽에 연결된 이웃을 인정하되 직접 상호 참조는 거부
private Collider FindConnection(RaycastHit[] hits)
{
    foreach (var hit in hits)
    {
        var other = hit.collider;
        if (hit.distance > 0.001f || other.CompareTag("Ice")
            || other.transform == transform) continue;
        bool movable = other.CompareTag("Spring") || other.CompareTag("Player1")
            || other.CompareTag("Player2");
        if (!movable) return other;

        var neighbor = other.GetComponent<WallConnection>();
        // 직접 벽에 닿지 않아도 이미 벽에 연결된 이웃을 통해 고정 상태를 전파한다.
        if (neighbor != null && neighbor.connectedToWall
            && !neighbor.ConnectedWith(GetComponent<Collider>()))
            return other;
    }
    return null;
}

// 이웃이 나를 직접 연결 대상으로 삼는지 검사해 양방향 참조를 피한다.
public bool ConnectedWith(Collider other)
    => other == _leftColl || other == _rightColl;

// 연결에 따른 축 제한. 다른 축의 중력·회전 제약은 유지한다.
public bool connectedToWall
{
    get => _connectedToWall;
    set
    {
        _connectedToWall = value;
        if (_mode != Mode.Spring) return;
        // 비트 연산으로 X축 제약만 켜고 꺼서 다른 물리 제약을 덮어쓰지 않는다.
        if (value) _rigid.constraints |= RigidbodyConstraints.FreezePositionX;
        else _rigid.constraints &= ~RigidbodyConstraints.FreezePositionX;
    }
}
```

```csharp
// PlayerJump.Landing: 벽과 겹친 BoxCast 결과를 바닥으로 오인하지 않는다.
private void UpdateLanding()
{
    if (Vector3.Dot(rigid.linearVelocity, customGravity.up) > 0f) return;
    var box = GameManager.instance.isSideView
        ? new Vector3(0.395f, 0.5f, 0f) : new Vector3(0.395f, 0f, 0.5f);
    var hits = Physics.BoxCastAll(rigid.position, box, customGravity.down,
        Quaternion.identity, 0.5f, LayerMask.GetMask("Platform"));
    isJumping = true;
    foreach (var hit in hits)
    {
        if (hit.collider.CompareTag("Spring") || hit.collider.CompareTag(tag))
            continue;
        if ((hit.collider.CompareTag("Player1") || hit.collider.CompareTag("Player2"))
            && !hit.collider.GetComponent<Movable>().isPaused) continue;

        // 바닥 normal은 중력 반대 방향과 유사하지만 옆벽 normal은 그렇지 않다.
        float alignment = Vector3.Dot(hit.normal, -customGravity.down.normalized);
        if (hit.distance < 0.07f && alignment > 0.9f)
        {
            isJumping = false;
            return;
        }
    }
}
```

## 생략 및 재구성한 내용

- 원본: `Assets/Scripts/Player/WallConnection.cs`·`PlayerJump.cs`, `Player.cs`·`SpringMove.cs`·`Movable.cs`. 기준 커밋: `0607414498b6af712b2c6225317c23dc2fa48ac7`.
- 좌우 탐색 결과를 Collider로 바로 반환하도록 줄였습니다. 기존 연결을 이웃에 전파하는 방식이며, 전체 그래프를 한 번에 탐색하거나 임의 길이의 순환을 검출하는 알고리즘은 아닙니다.
- 플레이어가 스프링을 미는 방향에 따른 연결 해제, 충돌 종료 후 갱신 재개, 빙판 가속·소리·애니메이션은 생략했습니다. 
- UpdateLanding은 InvokeLanding과 Landing을 합친 것입니다.
- WallConnection과 PlayerJump는 서로 다른 클래스의 발췌입니다. Mode·필드·Unity Rigidbody·Collider·태그·레이어는 원본 의존성으로 단독 실행되지 않습니다.
