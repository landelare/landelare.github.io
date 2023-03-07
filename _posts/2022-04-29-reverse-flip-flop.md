---
title:  "Reverse Flip Flop"
excerpt: "A small, but useful BP trick."
last_modified_at: 2023-03-07
---

All too often I see people copypasting entire blocks of BP just because they
need to change some parameters and there's nothing to `Select`.

Introducing the `Merge` node, which is as useful as it is simple to implement:

![Merge node in BP](/images/k2_merge.png)

# BP macro

![Merge macro in BP](/images/k2_merge_macro.png)

# BP macro with built-in Select

![Merge macro in BP with integrated Select](/images/k2_merge_macro_wildcard.png)

# C++ (sans wildcards)

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
