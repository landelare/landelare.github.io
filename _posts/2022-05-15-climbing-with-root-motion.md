---
title:  "Climbing with root motion"
excerpt: "Some ideas on how to dynamically adapt climbing animations."
---

[Advanced Locomotion System](https://www.unrealengine.com/marketplace/en-US/product/advanced-locomotion-system-v1)
on the Marketplace has some really cool basic locomotion features that can serve
as an inspiration for your own game.
In this article we'll look a little into its "Mantle System".

`ALS_Base_CharacterBP` has a few functions under the "Mantle System" category.
Of note is `MantleStart` which fetches a hand-authored curve that's being used
later to calculate where to start the climbing montage. This is super clunky and
requires you to keep these curves in sync with any animations that you might
make or adjust, and it doesn't even contain the root motion fully—you need a
`Mantle_Asset` for its multipliers.
It also leads to your character being "magically" dragged up as needed
(climb the 2.5M block at the back of the ALS test level!).

What if we used root motion for root motion? The idea is really simple: read
root motion from the animation directly and start the animation at an
appropriate point so that the _remaining_ root motion gets you where you want to
be.

Obviously this approach is not without its faults either—anticipation needs
extra thought since just finding a start point at the correct Z height will
often mean you want to start the animation mid-climb (which can still work out
for unexpectedly grabbing ledges mid-fall). The code as presented will only deal
with starting the animation from a time offset: correcting for XY, rotation, IK,
or blending anticipation is out of scope for this article. It is certainly
doable though, and is being done in the project where this code was originally
written as part of a larger system.

# Preparations

`ALS_N_Mantle_2m` is a good example animation if you need something for testing.
ALS itself is permanently free in the Marketplace, link near the top of this
page. You'll need to modify a few options to make this animation use the root
motion that's otherwise ignored by ALS: make sure root motion is **enabled**
and Force Root Lock is **disabled**.
We'll also be using a montage of this animation, play it in a full body slot to
get started.

Code snippets are written for a component that's attached to a character, but
there's nothing in them that would force you to use a component.

# Start!

Finding out if/where to climb is out of scope for this article, use ALS for
inspiration if you need any. We'll assume you've already committed to climbing
and you know the final transform after it's done.

Here's how to change properties on `ACharacter` to make the animation play well.
If you're using BSPs, note that they're neither actors nor components and will
come back as `nullptr` from collision checks. All code below is BSP aware.

```c++
// Climb this (usually static mesh) component to end up at Target
void UMyClimbComponent::StartClimbing(UPrimitiveComponent* Component,
                                      const FTransform& Target)
{
    // Attach the owning character to this component.
    // This is needed for moving/rotating platforms.
    FAttachmentTransformRules Rules(EAttachmentRule::KeepWorld, false);
    Character->AttachToComponent(Component, Rules);

    // Collisions were predicted when Target was calculated.
    // Disable them now because the character capsule will
    // pass through Component or BSP.
    CharacterCapsule->SetCollisionEnabled(ECollisionEnabled::NoCollision);

    // This makes Z root motion work.
    CharacterMovement->SetMovementMode(EMovementMode::MOVE_Flying);

    double CharacterZ = CharacterMovement->GetActorFeetLocation().Z;
    double DeltaZ = Target.GetTranslation().Z - CharacterZ;

    // We'll write this next.
    float StartTime = FindMontageStartForDeltaZ(DeltaZ);

    // Keeping play rate at 1 for simplicity but you can of course alter it.
    CharacterAnimInstance->Montage_Play(YourClimbMontage, 1,
        EMontagePlayReturnType::MontageLength, StartTime);
}

// Up to you to make sure this gets called - use an anim notify, sign up for a
// montage end delegate, make StartClimbing co_await the montage, etc.
void UMyClimbComponent::EndClimbing()
{
    FDetachmentTransformRules Rules(EDetachmentRule::KeepWorld, false);
    Character->DetachFromActor(Rules);
    CharacterCapsule->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
    CharacterMovement->SetMovementMode(EMovementMode::MOVE_Walking);
}
```

# Finding the start time

Root motion Z is assumed to broadly go from minimum to maximum height. It does
not need to be linear or even a well-behaved easing curve, but if you want to
have something wilder like a bounce you'll need to incorporate extra tiebreaker
logic to this code since the same Z height will be found at multiple times.

```c++
// Finds the point in the animation with the given amount of ΔZ left.
float UMyClimbComponent::FindMontageStartForDeltaZ(double DeltaZ) const
{
    FCompositeSection& Section = ClimbMontage->GetAnimCompositeSection(0);
    auto* Sequence = CastChecked<UAnimSequence>(Section.GetLinkedSequence());

    // Do a binary search - halve the search interval until we get close enough. 
    float Start = 0;
    float End = Sequence->GetPlayLength();

    FTransform EndTransform = Sequence->ExtractRootMotion(0, End, false);
    double TargetZ = EndTransform.GetTranslation().Z - DeltaZ;

//  Alternatively, use a for loop to limit the number of iterations if you
//  prefer a defensive coding style:
//  for (int i = 0; i < 10; ++i)
    while (true)
    {
        float T = (Start + End) / 2.0f;
        FTransform Transform = Sequence->ExtractRootMotion(0, T, false);
        double ZAtT = Transform.GetTranslation().Z;
    
        if (FMath::IsNearlyEqual(ZAtT, TargetZ, HeightTolerance))
            return T;
    
        if (ZAtT < TargetZ)
            Start = T;
        else
            End = T;
    }
}
```
