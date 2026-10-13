# Mock tests (interaction tests, C++)

## What

A test that replaces a collaborator with a **mock** and checks the **interaction**: which calls, with which arguments, how often, in which order.

## Why

- Some requirements **are** interactions. Example: "when the error is confirmed, send exactly one alarm to the alarm service."
- It lets you test code that depends on hardware, an OS, or a service that is not available.
- It lets you force rare answers from a collaborator (errors, timeouts) with no real fault.

## When

Use a mock when **the call itself is the requirement**, or when you must force a collaborator to fail on demand.

Do **not** use a mock when:

- you can check the **result** (state) instead. Prefer a **fake** or a **stub**.
- the type is a **value type** or a simple data holder.
- you **do not own** the type (wrap it in an adapter first).
- the check would repeat the production code step by step.

## How (GoogleMock)

1. Define an **interface** with a virtual destructor.
2. Create a mock class with `MOCK_METHOD`.
3. Inject the mock through the constructor.
4. Set `EXPECT_CALL` **before** the Act.
5. Do the Act. The mock checks the expectations when it is destroyed.

```cpp
class IAlarmSink {
public:
    virtual ~IAlarmSink() = default;
    virtual void Send(const AlarmRequest& request) = 0;
};

class MockAlarmSink : public IAlarmSink {
public:
    MOCK_METHOD(void, Send, (const AlarmRequest& request), (override));
};

// Verifies: REQ-XXX
TEST_F(AlarmManagerTest, ConfirmedError_SendsExactlyOneAlarm)
{
    ::testing::StrictMock<MockAlarmSink> sink;
    AlarmManager manager{sink, clock_};

    EXPECT_CALL(sink, Send(::testing::Field(&AlarmRequest::reason, AlarmReason::kLinkLost)))
        .Times(1);

    manager.OnErrorConfirmed(ErrorId::kLinkLost);
    manager.Step();
    manager.Step();   // second step must not send again
}
```

## Best practices

- **Prefer stub or fake first.** A mock is the last choice.
- **Mock roles, not classes.** Small interface, one role.
- **Use `StrictMock`** when an extra call is an error (outputs where an extra call is harmful). Use `NiceMock` when you only care about one call. Do not leave default (naggy) mocks in committed tests.
- **`ON_CALL`** sets default answers (stub). **`EXPECT_CALL`** sets requirements. Do not use `EXPECT_CALL` for things that are not requirements.
- **Check only what the requirement says.** Do not check the call order or count unless it matters (`InSequence`, `.Times`).
- **Use matchers** (`Field`, `Property`, `Pointee`, `DoubleNear`) instead of full-object equality.
- **Verify the effect as well**, when there is one. A call check alone does not show that the system works.
- **Force failures** with `WillOnce(Return(error))` or `Throw(...)`. Test that the unit handles them.
- **Mock only at boundaries**: hardware, OS, network, clock, other processes.
- **Do not mock** the unit under test, value types, `std::` types, or private functions.
- **Keep mocks in the test-utility library** when many tests use them.
- **Run contract tests** on the real implementation and on the fake, so the double matches reality.

## How to inject a mock

Use an interface, a template seam, a callable, or a link seam. Choice and cost: `../foundations/test-doubles.md`. Use `MockFunction<R(Args...)>` to check that a single callback was called. Wrap free and system functions in an interface. Do not mock global functions with hacks.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Test is mostly mock setup | Use a fake. Redesign the unit. |
| Break on refactoring | Check the result, not the steps |
| Mock returns a mock | Too much coupling. Simplify. |
| Expectation set after the Act | Set before |
| Mock leak (`new` mock never deleted, so no verification) | Use stack objects or `std::unique_ptr` |
| `StrictMock` on a chatty interface | Use `NiceMock` and set one `EXPECT_CALL` |
| Mock does not match real behavior | Add a contract test |

## Checklist

- [ ] The interaction is a real requirement.
- [ ] Strict or nice mock chosen on purpose.
- [ ] Only needed calls and arguments are checked.
- [ ] Failure answers from the collaborator are tested.
- [ ] The effect is also checked, when there is one.
