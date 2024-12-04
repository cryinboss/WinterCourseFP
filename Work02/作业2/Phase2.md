# 2024/12/2

主要解决问题：

- 射击命中方块，获得积分X分；
- 设计超过一定次数后，方块销毁；
- GameState设计与维护
- PlayerState设计与维护

GameState主要维护所有玩家的PlayerState。

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "MyPlayerState.h"
#include "fps02/fps02PlayerController.h"
#include "GameFramework/GameStateBase.h"
#include "MyGameState.generated.h"

/**
 * 
 */
UCLASS()
class FPS02_API AMyGameState : public AGameStateBase
{
	GENERATED_BODY()
public:
	AMyGameState();
	virtual void BeginPlay() override;
	virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	virtual void Tick(float DeltaSeconds) override;
	 UFUNCTION()
	 void PlayerInfo();
	UPROPERTY(VisibleAnywhere,BlueprintReadOnly)
	TArray<TObjectPtr<AMyPlayerState>> AllPlayersState;
	void AddPlayerState(APlayerState* PlayerState);
	
};

```

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#include "MyGameState.h"

#include "fps02/fps02PlayerController.h"

AMyGameState::AMyGameState()
{
}

void AMyGameState::BeginPlay()
{
	Super::BeginPlay();
}

void AMyGameState::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
	Super::EndPlay(EndPlayReason);
}

void AMyGameState::Tick(float DeltaSeconds)
{
	Super::Tick(DeltaSeconds);
	
}

void AMyGameState::PlayerInfo()
{
	for(TObjectPtr<AMyPlayerState> PlayerState:AllPlayersState)
	{
		if (PlayerState)
		{
			UE_LOG(LogTemp, Warning, TEXT("Player Name: %s"), *PlayerState->GetName());
		}
	}
}

void AMyGameState::AddPlayerState(APlayerState* PlayerState)
{
	if (PlayerState && !AllPlayersState.Contains(PlayerState))
	{
		if (TObjectPtr<AMyPlayerState> CastedState = Cast<AMyPlayerState>(PlayerState))
		{
			AllPlayersState.Add(CastedState);
		}
	}
}
```

每个人的玩家状态，这里写代码的时候发现Score是父类就定义好的变量。

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "MyPlayerState.generated.h"

/**
 * 
 */
UCLASS()
class FPS02_API AMyPlayerState : public APlayerState
{
	GENERATED_BODY()
	
public:
	AMyPlayerState();
	UFUNCTION(Server, Reliable, WithValidation)
	void ServerUpdateScore(int32 ScoreChange);
	virtual void OnRep_Score() override;
	void UpDateScore(int32 ScoreChange);
	virtual void BeginPlay() override;
};
```

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#include "MyPlayerState.h"

#include "Net/UnrealNetwork.h"

AMyPlayerState::AMyPlayerState()
{
}
void AMyPlayerState::ServerUpdateScore_Implementation(int32 ScoreChange)
{
	UpDateScore(ScoreChange);
}

bool AMyPlayerState::ServerUpdateScore_Validate(int32 ScoreChange)
{
	return true;
}

void AMyPlayerState::OnRep_Score()
{
	Super::OnRep_Score();
	UE_LOG(LogTemp, Warning, TEXT("玩家%s 当前得分:%f"),*GetPlayerName(),GetScore());
}

void AMyPlayerState::UpDateScore(int32 ScoreChange)
{
	SetScore(GetScore()+ScoreChange);
	OnRep_Score();
}

void AMyPlayerState::BeginPlay()
{
	Super::BeginPlay();
}

```

NetConnection直接连着每个玩家的PlayerController，因此在GameMode里设立监听事件，PostLogin确保玩家已经成功加入游戏，网络连接已建立。只要有玩家加入游戏就将其玩家状态加入游戏状态管理集。

```cpp
void AMyGameMode::PostLogin(APlayerController* NewPlayer)
{
	Super::PostLogin(NewPlayer);
	TObjectPtr<Afps02PlayerController> Player=Cast<Afps02PlayerController>(NewPlayer);
	if(Player)
	{
		AMyGameState* MyGameState = GetWorld()->GetGameState<AMyGameState>();
		TObjectPtr<AMyPlayerState> MyPlayerState=Player->GetPlayerState<AMyPlayerState>();
		if(MyPlayerState)
		{
			MyGameState->AddPlayerState(MyPlayerState);
		}
	}
}
```

子弹OnHit函数逻辑完善，命中之后，获取方块Actor，先处理方块血量（这里伤害和命中次数相同=1），由于Health变量已经处理为网络复制，因此自动同步。当方块血量≤0，进入销毁逻辑。服务端有Authority权限直接销毁，客户端使用远程调用Server RPC的方式由服务端负责统一销毁。

```cpp
void Afps02Projectile::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp,
                             FVector NormalImpulse, const FHitResult& Hit)
{
	if ((OtherActor != nullptr) && (OtherActor != this) && (OtherComp != nullptr) && OtherComp->IsSimulatingPhysics())
	{
		OtherComp->AddImpulseAtLocation(GetVelocity() * 50.0f, GetActorLocation());
		
		if (OtherActor->IsA(AMyCubeActor::StaticClass()))
		{
			TObjectPtr<AMyCubeActor> MyCube = Cast<AMyCubeActor>(OtherActor);
			if (MyCube)
			{
				MyCube->SetHealth(MyCube->Health()-1);
				
				TObjectPtr<Afps02Character> MyPawn = Cast<Afps02Character>(GetOwner());
				
				if (MyPawn)
				{
					TObjectPtr<Afps02PlayerController> MyPlayerController = Cast<Afps02PlayerController>(
						MyPawn->GetController());
					if (MyPlayerController)
					{
						TObjectPtr<AMyPlayerState> MyPlayerState = Cast<
							AMyPlayerState>(MyPlayerController->GetPlayerState<AMyPlayerState>());

						if (MyPlayerState)
						{
							if (MyPawn->HasAuthority())
							{

								if(MyCube->Health()<=0)
								{
								 MyPlayerState->UpDateScore(MyCube->Score());
									MyCube->DestroyCube();
								}
								
							}
							else
							{
								
								if(MyCube->Health()<=0)
								{
								MyPlayerState->ServerUpdateScore(MyCube->Score());
									MyPlayerController->ServerHandleCubeDamage(MyCube);
								}
							}
						}
					}
				}
			}
		}
		Destroy();
	}
}
```

Server RPC写在玩家控制器里：

```cpp
	UFUNCTION(Server, Reliable, WithValidation)
	void ServerHandleCubeDamage(AMyCubeActor* CubeActor);
```

加上实现和验证：

```cpp
void Afps02PlayerController::ServerHandleCubeDamage_Implementation(AMyCubeActor* CubeActor)
{
	if(CubeActor&&CubeActor->Health()<=0)
	{
		CubeActor->DestroyCube();
	}
}

bool Afps02PlayerController::ServerHandleCubeDamage_Validate(AMyCubeActor* CubeActor)
{
	return CubeActor != nullptr;
}
```