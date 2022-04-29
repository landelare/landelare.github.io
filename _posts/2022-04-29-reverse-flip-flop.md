---
title:  "Reverse Flip Flop"
excerpt: "A small, but useful BP trick."
---

All too often I see people copypasting entire blocks of BP just because they
need to change some parameters and there's nothing to `Select`.

Introducing the `Merge` node, which is as useful as it is simple to implement:

![Merge node in BP](/images/k2_merge.png)

```c++
UENUM()
enum class EMyAB : uint8
{
    A,
    B,
};

UCLASS()
class MY_API UMyLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Meta = (ExpandEnumAsExecs = "Exec"))
    static UPARAM(DisplayName = "A") bool Merge(EMyAB Exec)
    {
        return Exec == EMyAB::A;
    }
};
```

It's also possible to implement entirely in BP as a macro:

![Merge macro in BP](/images/k2_merge_macro.png)
