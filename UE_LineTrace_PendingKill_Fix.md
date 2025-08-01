# Unreal Engine Line Trace "Pending Kill or Garbage" Error Fix

## Problem Description
You're getting the error: **"Attempted to access but is not valid (pending kill or garbage)"** when using line traces in Unreal Engine. This typically happens when:
- Trying to access a hit actor that has been destroyed
- Accessing objects without proper validation
- Objects are being garbage collected while you're trying to use them

## Root Causes
1. **Hit Actor Destroyed**: The actor hit by the line trace was destroyed before you accessed it
2. **Missing UPROPERTY**: Object references without UPROPERTY can be garbage collected
3. **No Validation**: Not checking if objects are valid before accessing them
4. **Timing Issues**: Accessing objects during destruction or garbage collection

## Solutions

### 1. **Always Use IsValid Check** (Most Important)

**In Blueprints:**
```blueprint
Line Trace by Channel → Hit Actor → IsValid? → [Valid] Your Logic
                                           → [Invalid] Handle Error/Skip
```

**In C++:**
```cpp
FHitResult HitResult;
if (GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECC_Visibility))
{
    if (IsValid(HitResult.GetActor()))
    {
        // Safe to access the hit actor
        AActor* HitActor = HitResult.GetActor();
        // Your logic here
    }
    else
    {
        // Hit actor is invalid/destroyed - handle gracefully
        UE_LOG(LogTemp, Warning, TEXT("Hit actor is invalid"));
    }
}
```

### 2. **Proper Object Reference Management**

**For Blueprint Variables:**
- Use `UPROPERTY()` for any actor/object references in C++
- In Blueprints, variables are automatically UPROPERTYs

**C++ Example:**
```cpp
UCLASS()
class YOURGAME_API AYourActor : public AActor
{
    GENERATED_BODY()

private:
    // CORRECT: Will not be garbage collected
    UPROPERTY()
    AActor* CachedHitActor;

    // WRONG: Can be garbage collected unexpectedly
    AActor* BadReference;
};
```

### 3. **Validated Get for Variables**

**In Blueprints:**
- Right-click on your actor variable
- Select "Convert to Validated Get"
- This automatically adds IsValid checking

### 4. **Safe Line Trace Pattern**

**Blueprint Implementation:**
```blueprint
1. Line Trace by Channel
2. Break Hit Result
3. IsValid? (Hit Actor)
   ├─ Valid → Continue with your logic
   └─ Invalid → Print warning or handle error
```

**C++ Safe Pattern:**
```cpp
void AYourActor::SafeLineTrace()
{
    FVector Start = GetActorLocation();
    FVector End = Start + (GetActorForwardVector() * 1000.0f);
    
    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);
    
    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        Start,
        End,
        ECC_Visibility,
        QueryParams
    );
    
    if (bHit)
    {
        // Always validate before accessing
        if (IsValid(HitResult.GetActor()))
        {
            AActor* HitActor = HitResult.GetActor();
            
            // Additional component validation if needed
            if (UStaticMeshComponent* MeshComp = HitActor->FindComponentByClass<UStaticMeshComponent>())
            {
                if (IsValid(MeshComp))
                {
                    // Safe to use the component
                    MeshComp->SetMaterial(0, NewMaterial);
                }
            }
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Hit actor is pending kill or invalid"));
        }
    }
}
```

### 5. **Handle Delays and Timers Safely**

**Problem:** Object gets destroyed during a delay/timer

**Solution:**
```blueprint
Line Trace → Store Hit Actor in Variable → Delay → IsValid? → Use Actor
```

**C++ Timer Example:**
```cpp
void AYourActor::LineTraceWithDelay()
{
    // Perform line trace
    FHitResult HitResult;
    if (GetWorld()->LineTraceSingleByChannel(HitResult, Start, End, ECC_Visibility))
    {
        // Store weak reference to prevent garbage collection issues
        TWeakObjectPtr<AActor> WeakHitActor = HitResult.GetActor();
        
        // Use timer with lambda
        GetWorld()->GetTimerManager().SetTimer(
            TimerHandle,
            [this, WeakHitActor]()
            {
                if (WeakHitActor.IsValid())
                {
                    // Safe to access
                    AActor* HitActor = WeakHitActor.Get();
                    // Your delayed logic here
                }
                else
                {
                    UE_LOG(LogTemp, Warning, TEXT("Actor was destroyed during delay"));
                }
            },
            2.0f,
            false
        );
    }
}
```

### 6. **Weak Object Pointers (C++)**

Use `TWeakObjectPtr` for references that might become invalid:

```cpp
UCLASS()
class YOURGAME_API AYourActor : public AActor
{
private:
    // Won't prevent garbage collection, but safely becomes null
    TWeakObjectPtr<AActor> WeakActorReference;

public:
    void StoreHitActor(AActor* HitActor)
    {
        WeakActorReference = HitActor;
    }
    
    void UseStoredActor()
    {
        if (WeakActorReference.IsValid())
        {
            AActor* Actor = WeakActorReference.Get();
            // Safe to use
        }
        else
        {
            // Actor was destroyed
            UE_LOG(LogTemp, Warning, TEXT("Stored actor was destroyed"));
        }
    }
};
```

## Common Scenarios and Solutions

### Scenario 1: Line Trace in Tick
```cpp
// WRONG: No validation
void AYourActor::Tick(float DeltaTime)
{
    FHitResult Hit;
    if (GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility))
    {
        Hit.GetActor()->Destroy(); // Can cause pending kill error
    }
}

// CORRECT: With validation
void AYourActor::Tick(float DeltaTime)
{
    FHitResult Hit;
    if (GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility))
    {
        if (IsValid(Hit.GetActor()))
        {
            Hit.GetActor()->Destroy();
        }
    }
}
```

### Scenario 2: Multiple Line Traces
```blueprint
1. First Line Trace → Hit Actor A → Store in Variable
2. Second Line Trace → Hit Actor B
3. IsValid? (Actor A) → Use Actor A
4. IsValid? (Actor B) → Use Actor B
```

### Scenario 3: Component Access
```cpp
if (IsValid(HitResult.GetActor()))
{
    if (UMeshComponent* MeshComp = HitResult.GetComponent())
    {
        if (IsValid(MeshComp))
        {
            // Safe to use component
            MeshComp->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        }
    }
}
```

## Prevention Best Practices

1. **Always validate before accessing**: Use `IsValid()` before any object access
2. **Use UPROPERTY**: For any stored object references in C++
3. **Weak pointers**: Use `TWeakObjectPtr` for optional references
4. **Validated Get**: Convert Blueprint variables to Validated Get
5. **Handle destruction gracefully**: Don't assume objects persist
6. **Avoid storing raw pointers**: Prefer UPROPERTY or weak pointers

## Debugging Tips

1. **Enable logging** to see when objects are destroyed:
```cpp
UE_LOG(LogTemp, Warning, TEXT("Actor %s is being accessed but is invalid"), 
       HitResult.GetActor() ? *HitResult.GetActor()->GetName() : TEXT("NULL"));
```

2. **Check Blueprint references** in the details panel
3. **Use breakpoints** before accessing hit actors
4. **Monitor garbage collection** with `stat gc` console command

## Quick Checklist
- [ ] Using `IsValid()` before accessing hit actors?
- [ ] Object references marked with `UPROPERTY()`?
- [ ] Handling delays/timers with validation?
- [ ] Using Validated Get for Blueprint variables?
- [ ] Not storing raw pointers to actors?

This error is completely preventable with proper validation and reference management!