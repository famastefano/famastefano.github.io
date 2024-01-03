---
title:  "Unit Testing AActor and AActorComponent"
excerpt: "How to setup the test environment to easily test our Actors and their Components"
---

## Abstract

After I completed my first draft of an `ASceneComponent`, I wanted to test it properly like I'm used to with frameworks like Google Test and Catch2.

To do so, I had to instantiate and initialize a World, ready to be used by my Actors.

To give you an idea on what I was able to do, here's one of the tests I wrote:
```cpp
It("Should allow to fire again if enough time has passed", [this]
{
    FComponentOptions Opt;
    Opt.FireRateRpm = 600;
    Opt.HasInfiniteAmmo = true;
    Opt.AmmoType.IsHitScan = true;
    // Instantiate the component to test
    auto* Component = CreateAndAttachComponent(Opt);
    const double WaitingTimeBetweenEachShot = Component->GetSecondsBetweenShots() + 0.1;
    Component->FireOnce();
    
    // Here we tick our world created ad-hoc for our tests
    World.Tick(WaitingTimeBetweenEachShot);
    Component->FireOnce();
    
    // We can choose a custom DeltaTime
    World.Tick(WaitingTimeBetweenEachShot);
    Component->FireOnce();

    // Use the default DeltaTime if we don't care
    World.Tick();
    TestTrueExpr(DelegateHandler->OnShotFiredCounter == 3);
});
```

## Steps

My current design consists in a module that:
* Uses a `UEngineSubsystem` to create/initialize/destroy the `UWorld` instances.
* A `FTestWorldHelper` that acts like a `unique_ptr` for our Worlds.
* A custom `UGameInstance` that registers itself with the new World.

To simplify this post, the method's implementation will be provided inline, but you need to split it into header/source, expecially for the `FTestWorldHelper` and the `Subsystem` as there's a recursive include otherwise.

### Create a `UGameInstance` and `AGameModeBase`

Why do we need these? Look at these lines of `UWorld::BeginPlay`:
```cpp
AGameModeBase* const GameMode = GetAuthGameMode(); // this just returns AuthorityGameMode
...
if (GameMode)
{
    GameMode->StartPlay();
    ...
}
...
```
`AGameModeBase::StartPlay` will eventually call `AActor::BeginPlay` for each Actor loaded in the World, and because `UWorld::HasBegunPlay` will return true, newly spawned Actors will get their `BeginPlay` called too.

But when does a `UWorld` acquire get a `GameMode`? Here:
```cpp
bool UWorld::SetGameMode(const FURL& InURL)
{
    ...
    AuthorityGameMode = GetGameInstance()->CreateGameModeForURL(InURL, this);
    ...
}
```
`UGameInstance::CreateGameModeForURL` creates and initializes the correct `AGameModeBase` for *that level*.

So, to complete this step, let's create an empty `AGameModeBase`, that I called `ATestWorldGameMode`.

Then, our Game Instance, `UTestWorldGameInstance`:
```cpp
UCLASS()
class TESTWORLD_API UTestWorldGameInstance : public UGameInstance
{
	GENERATED_BODY()

public:
	void InitForTest(UWorld& World)
    {
        FWorldContext* TestWorldContext = GEngine->GetWorldContextFromWorld(&World);
        check(TestWorldContext);
        WorldContext = TestWorldContext;
        WorldContext->OwningGameInstance = this;
        World.SetGameInstance(this);
        World.SetGameMode(FURL()); // Now the UWorld::AuthorityGameMode will be valid

        Init();
    }

    // We don't care which game mode the base class has created, we always return ours.
	virtual TSubclassOf<AGameModeBase> OverrideGameModeClass(
        TSubclassOf<AGameModeBase> GameModeClass,
	    const FString& MapName,
        const FString& Options,
	    const FString& Portal) const override
        {
            return ATestWorldGameMode::StaticClass();
        }
};
```

### Create a helper class to manage `UWorld` ticking

> *Why do I need a wrapper for `UWorld`? Can't I just call `UWorld::Tick`?*

Nope. *Turns out* that there's plenty of stuff under the hood that is required to actually tick properly.

By searching all references of `UWorld::Tick` you'll end up here:
```cpp
void UGameEngine::Tick( float DeltaSeconds, bool bIdleMode )
{
    ...
    // Tick the world.
    Context.World()->Tick( LEVELTICK_All, DeltaSeconds );
    ...
}
```
There's a lot of stuff that is "ticked" before and after that, so to actually do a proper tick we need to:
```cpp
// -- Extracted from UGameEngine::Tick

// Update subsystems.
// This assumes that UObject::StaticTick only calls ProcessAsyncLoading.
StaticTick(DeltaSeconds, !!GAsyncLoadingUseFullTimeLimit, GAsyncLoadingTimeLimit / 1000.f);
// Tick the world.
Context.World()->Tick( LEVELTICK_All, DeltaSeconds );
// Tick all tickable objects
FTickableGameObject::TickObjects(nullptr, LEVELTICK_All, false, DeltaTime);

// -- Extracted from FEngineLoop::Tick

// Increase the frame counters, otherwise Actors and Components will not tick again!
GFrameCounter++;
// need to process gamethread tasks at least once a frame no matter what
FTaskGraphInterface::Get().ProcessThreadUntilIdle(ENamedThreads::GameThread);

// tick core ticker, threads & deferred commands
FThreadManager::Get().Tick();
FTSTicker::GetCoreTicker().Tick(DeltaTime);
GEngine->TickDeferredCommands();
```

This doesn't include *everything*, `Slate` isn't ticked, viewport isn't updated, seamless travel and async loading aren't updated either etc.

I don't need them, but if you need do, look into `FEngineLoop::Tick`, `UGameEngine::Tick` and `UWorld::Tick`.

Now, let's create a C++ class `FTestWorldHelper`:
```cpp
#include "Engine/CoreSettings.h"
#include "HAL/ThreadManager.h"

class UTestWorldSubsystem;

class TESTWORLD_API FTestWorldHelper
{
    // Subsystem that manages the created Worlds
	UTestWorldSubsystem* Subsystem;
	UWorld* World;
    // If NOT shared, then we need to cleanup it when we are destroyed
	bool IsSharedWorld;
	decltype(GFrameCounter) OldGFrameCounter;

public:
	explicit FTestWorldHelper() :
		Subsystem(nullptr),
		World(nullptr),
		IsSharedWorld(false),
		OldGFrameCounter(GFrameCounter)
	{
	}

	explicit FTestWorldHelper(UTestWorldSubsystem* Subsystem,
	                          UWorld* World,
	                          bool IsSharedWorld)
    : Subsystem(Subsystem),
      World(World),
      IsSharedWorld(IsSharedWorld),
      OldGFrameCounter(GFrameCounter)
    {}

    // This class is like std::unique_ptr for UWorld instances
	FTestWorldHelper(const FTestWorldHelper&) = delete;
	FTestWorldHelper& operator=(const FTestWorldHelper&) = delete;

	FTestWorldHelper(FTestWorldHelper&& Other) noexcept;
	FTestWorldHelper& operator=(FTestWorldHelper&& Other) noexcept;

	~FTestWorldHelper()
    {
        // World created just for us, so destroy it as we don't need it anymore
        if (World && !IsSharedWorld)
        {
            GFrameCounter = OldGFrameCounter;
            Subsystem->DestroyPrivateWorld(World->GetFName());
        }
    }

	FORCEINLINE UWorld* operator->() const
	{
		check(World);
		return World;
	}

    // Smallest DeltaTime with an exact representation.
    // Personally I needed such a small default.
	void Tick(float DeltaTime = 0.001953125) const
    {
        check(IsInGameThread());
        check(World);
        StaticTick(DeltaTime, !!GAsyncLoadingUseFullTimeLimit, GAsyncLoadingTimeLimit / 1000.f);
        World->Tick(LEVELTICK_All, DeltaTime);
        FTickableGameObject::TickObjects(nullptr, LEVELTICK_All, false, DeltaTime);
        GFrameCounter++;
        FTaskGraphInterface::Get().ProcessThreadUntilIdle(ENamedThreads::GameThread);
        FThreadManager::Get().Tick();
        FTSTicker::GetCoreTicker().Tick(DeltaTime);
        GEngine->TickDeferredCommands();
    }
};
```

### Create the `UEngineSubsystem` that manages the `UWorld`

Why an Engine Subsystem? Well personally I wanted a Singleton that were available anywhere in my tests, and because I didn't want to initialize a new `UWorld` each time.

Let's start by creating a `UTestWorldSubsystem`:
```cpp
USTRUCT()
struct FTestWorldData
{
	GENERATED_BODY()

	UPROPERTY()
	TObjectPtr<UGameInstance> GameInstance;

	UPROPERTY()
	TObjectPtr<UWorld> World;
};

UCLASS()
class TESTWORLD_API UTestWorldSubsystem : public UEngineSubsystem
{
	GENERATED_BODY()

	TMap<FName, FTestWorldData> PrivateWorlds;

	FTestWorldData SharedWorld;

	static FTestWorldData MakeTestWorld(FName Name);

public:
	virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
    }

	virtual void Deinitialize() override
    {
        // Dispose of the created Worlds
        if (!PrivateWorlds.IsEmpty())
        {
            for (const auto& [Name, Env] : PrivateWorlds)
            {
                // This shouldn't happen, might want to log a warning
                Env.World->DestroyWorld(true);
                Env.GameInstance->RemoveFromRoot();
            }
            PrivateWorlds.Empty();
        }

        if (SharedWorld.World)
        {
            SharedWorld.World->DestroyWorld(true);
            SharedWorld.GameInstance->RemoveFromRoot();
        }

        Super::Deinitialize();
    }

    FTestWorldHelper UTestWorldSubsystem::GetPrivateWorld(FName Name)
    {
        check(IsInGameThread());
        checkf(PrivateWorlds.Find(Name) == nullptr, TEXT("This test world has already been created"));

        const auto& [GameInstance, World] = PrivateWorlds.Add(Name, MakeTestWorld(Name));
        return FTestWorldHelper{this, World, false};
    }

    FTestWorldHelper UTestWorldSubsystem::GetSharedWorld()
    {
        // Lazy initialize the shared World,
        // because doing so in UEngineSubsystem::Initialize
        // is too early!
        if (!SharedWorld.World)
        {
            SharedWorld = MakeTestWorld("TestWorld_SharedWorld");
        }

        FTestWorldHelper Helper{this, SharedWorld.World, true};
        return Helper;
    }

	void DestroyPrivateWorld(FName Name)
    {
        const auto& [GameInstance, World] = PrivateWorlds.FindAndRemoveChecked(Name);
        World->DestroyWorld(true);
        GameInstance->RemoveFromRoot();
    }
};
```

Let's analyze how we create a test World:
```cpp
FTestWorldData UTestWorldSubsystem::MakeTestWorld(FName Name)
{
	check(IsInGameThread());

    // Create a Game World
	FWorldContext& WorldContext = GEngine->CreateNewWorldContext(EWorldType::Game);
	UWorld* World = UWorld::CreateWorld(EWorldType::Game, true, Name);
	UTestWorldGameInstance* GameInstance = NewObject<UTestWorldGameInstance>();
	GameInstance->AddToRoot(); // Avoids GC

	WorldContext.SetCurrentWorld(World);
	World->UpdateWorldComponents(true, true);
	World->AddToRoot();
	World->SetFlags(RF_Public | RF_Standalone);
    // Engine shouldn't Tick this UWorld.
	World->SetShouldTick(false);

    // Remember UTestWorldGameInstance::InitForTest?
	GameInstance->InitForTest(*World);

#if WITH_EDITOR
	GEngine->BroadcastLevelActorListChanged();
#endif

	World->InitializeActorsForPlay(FURL());
	auto* Settings = World->GetWorldSettings();
    // Unreal clamps the DeltaTime
	Settings->MinUndilatedFrameTime = 0.0001;
	Settings->MaxUndilatedFrameTime = 10;

    // Finally, we start playing
	World->BeginPlay();

	return {GameInstance, World};
}
```

*That's all folks!*

Our Worlds are ready to be used in our tests!

## Real World Usage

Here's an example on how I use this system:
```cpp
BEGIN_DEFINE_SPEC(FBallisticWeaponComponent_Spec, "WeaponSystemPlugin.Runtime.BallisticWeaponComponent",
                  EAutomationTestFlags::ApplicationContextMask
                  | EAutomationTestFlags::HighPriority | EAutomationTestFlags::ProductFilter)

	TObjectPtr<UTestWorldSubsystem> Subsystem;
	FTestWorldHelper World;
	TObjectPtr<AActor> Actor;
	TObjectPtr<UBallisticWeaponComponent> PrevComponent;

	struct FComponentOptions
	{
		...
	};

	UBallisticWeaponComponent* CreateAndAttachComponent(const FComponentOptions& Options = FComponentOptions())
	{
		...
		return PrevComponent;
	}

END_DEFINE_SPEC(FBallisticWeaponComponent_Spec)

void FBallisticWeaponComponent_Spec::Define()
{
	Describe("The Ballistic Weapon Component", [this]
	{
		BeforeEach([this]
		{
            // Get the subsystem once
			if (!Subsystem)
			{
				Subsystem = GEngine->GetEngineSubsystem<UTestWorldSubsystem>();
			}
            // Get our shared world, but could be private as well
			World = Subsystem->GetSharedWorld();

            // We can spawn actors!
			Actor = World->SpawnActor<ATestWorldActor>();
		});

		Describe("When trying to fire once", [this]
		{
			It("Cant fire because there is no ammo", [this]
			{
                // This creates a component and attaches it to the Actor.
                // UBallisticWeaponComponent::BeginPlay will be called!
				auto* Component = CreateAndAttachComponent();
				Component->FireOnce();

                // Tick our Test World, Actor::Tick and UBallisticWeaponComponent::TickComponent will be called!
				World.Tick();
				TestTrueExpr(DelegateHandler->OnShotFiredCounter == 0);
			});
        });

        AfterEach([this]{
            Actor->Destroy();
			Actor = nullptr;
			PrevComponent = nullptr;
        });
    });
}
```

## Considerations
* It's enough for my use case, and I've only tested it with a `USceneComponent`.
* We aren't ticking everything, like level streaming/transitioning etc. because I don't need it.
* For complex tests you might want to use Gauntlet, as it's a real running game, and provides more features.
* The current design might leak state due to the SharedWorld being, well, *shared*. Proper Setup/Cleanup in a test is required.
* The current design creates a dedicated GameInstance per World, further research is needed to see if we can use a single `UGameInstance` for multiple `UWorld` instances, but I didn't need such "optimization".

## Credits
I wouldn't have accomplished all of this without the help of:
* Unreal Slackers's Discord, `#cpp` channel!
* [`UE5Coro TestWorld`](https://github.com/landelare/ue5coro/blob/master/Plugins/UE5Coro/Source/UE5CoroTests/Private/TestWorld.cpp) credits to the repo owner.
* Epic Games for writing readable code :D