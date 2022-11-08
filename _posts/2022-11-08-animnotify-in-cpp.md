---
title: "AnimNotify in C++"
excerpt: "A lesser-known feature of UAnimInstance."
---

Not much this time, but apparently this is not widely known.
On your UAnimInstance subclass, a `UFUNCTION` named `AnimNotify_YourNotifyName`
will automatically get called when that notify happens.
These can in turn conveniently broadcast any delegates that you might want:

```c++
UCLASS()
class EXAMPLE_API UExampleAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable)
    FSomeDelegate OnFootstep;

protected:
    UFUNCTION()
    void AnimNotify_Footstep() const
    {
        OnFootstep.Broadcast();
    }
};
```
