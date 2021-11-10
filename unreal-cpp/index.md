# C++ na Unreal

Um guia de sobrevivência para programadores C++ em Unreal.

Lembrando que isso não substitui ler a documentação da Unreal (e às vezes até mesmo o código fonte também),
mas é um bom começo.

# Coisinhas de C++

## `#include "SomeHeaderFile.h"`
Para o compilador C++, um arquivo ter extensão `.h` ou `.cpp` ou `.macaco` não faz diferença.
O que acontece por trás dos panos é que o compilador vai processar individualmente (em geral em paralelo) cada
arquivo fonte que a gente passa pra ele (por convenção, os `.cpp`) que, por sua vez, pode incluir outros arquivos
(por convenção, os `.h`).

O que isso implica é que a princípio cada unidade de compilação são os `.cpp` individuais mais os `.h` que ele inclui.
Portanto, se a gente consegue reduzir o quanto de outros arquivos os `.h` incluem, menos retrabalho o compilador precisa fazer.

Então o truque para ter builds rápidas é abusar de _forward declarations_* e incluir o mínimo possível nos `.h`.

### _forward declaration_
É quando você define apenas o nome de um tipo (em geral classe) para usar um ponteiro daquele
tipo ao inves da declaração completa. Isso funciona porque para declarar um ponteiro, o compilador apenas precisa saber
o nome do tipo já que todos os ponteiros têm o mesmo tamanho.

Ex:
```cpp
// .h

// aqui a gente não inclui "BoxComponent.h"
// apenas declara seus ponteiros como
// `class UBoxComponent* MyVar;`

UCLASS()
class AMyActor : AActor
{
	GENERATED_BODY()
public:
	UPROPERTY(VisibleAnywhere)
	class UBoxComponent* PhysicsBodyComponent;
}

// .cpp
#include "MyActor.h"
#include "Components/BoxComponent.h"

// já no cpp precisamos incluir o arquivo
// pois provavelmente iremos usar algum método
// seu por aqui
```

### _static free functions_
Essas são funções que não pertencem a uma classe e que apenas existem em um `.cpp`.
Se você tem um método privado em sua classe que não precisa acessar variáveis privadas,
considere transformá-lo em uma _static free function_ que é menos código que precisa
ser declarado no `.h`.

Ex:
```cpp
// .h
UCLASS()
class AMyActor : AActor
{
	GENERATED_BODY()
public:
	UPROPERTY(EditAnywhere)
	FString MyName;
}

// .cpp
// função utilitária que só é usada nesse .cpp
// e apenas acessa campos que já são publicos
// poupa uma declaração privada que 'sujaria' no .h
static void PrintMyActorName(const AMyActor* Self)
{
	if (GEngine != nullptr)
	{
		GEngine->AddOnScreenDebugMessage(
			-1,
			0.0f,
			FColor::Red,
			Self->MyName
		);
	}
}

void AMyActor::BeginPlay()
{
	Super::BeginPlay();
	PrintMyActorName(this);
}
```

## Ponteiros*


# Coisinhas de Unreal

## `UPROPERTY()`
Marca um campo como uma propriedade que a Unreal entende.
Pode ter vários sub-atributos que extendem a semântica da propriedade.

```cpp
UPROPERTY()
UPrimitiveComponent* MyComponent;
```

Importante para campos que são ponteiros para `UObject` que ajuda o GarbageCollector da Unreal saber
quando ainda tem referencia para um determinado objeto.

### `UPROPERTY(Transient)`
Uma propriedade `Transient` significa que ela não é serializada. Isso é: ela não tem seu valor vindo do disco
(ou do editor por exemplo). Ao invés disso a Unreal assume que você sempre irá iniciar o seu valor manualmente.

Ex:
```cpp
// .h
// valor vem da blueprint que herda da classe
UPROPERTY(Transient)
UMovementComponent* MovementComponent;

// .cpp
void AMyActor::BeginPlay()
{
	Super::BeginPlay();

	// propriedade sempre inicializada
	MovementComponent = (UMovementComponent*)
		FindComponentByClass(UMovementComponent::StaticClass());
}
```

### `UPROPERTY(EditAnywhere)`
Permite uma propriedade ser editada tanto no editor como através de uma Blueprint.

### `UPROPERTY(VisibleAnywhere)`
Permite uma propriedade ser visualizada (somente leitura) tanto pelo editor como por uma Blueprint.

### `UPROPERTY(EditAnywhere, BlueprintReadOnly)`
Faz uma property ser editável no editor, porém uma Blueprint somente tem acesso de leitura.

## `UFUNCTION()`
Marca uma função para ser reconhecida pela Unreal.
Também pode ter vários sub-atributos que extendem a semântica da função.

### `UFUNCTION(BlueprintCallable)`
Permite uma função ser chamável por blueprint.

### `UFUNCTION(BlueprintPure)`
Permite uma função ser chamável por blueprint, porém como uma função que não tem efeitos colaterais (ex: muda estado, faz IO).
É recomendável que tal função seja marcada como `const`.

### `UFUNCTION(BlueprintImplementableEvent)`
Tais funções são implementadas *apenas* por Blueprints e nunca por C++.
Útil para definir eventos que são ouvidos apenas pela Blueprint que irá herdar de nossa classe C++.

### RPCs
RPCs são `UFUNCTION()` especiais que servem para trocar mensagens entre servidor e clientes.

Essas funções sempre são apenas declaradas no `.h` e você, na verdade, apenas implementa uma outra função
de mesmo nome porém que termina em `_Implementation`. Isso é porque a Unreal gera pra gente o corpo da
função original para enviar a mensagem pela rede. Ao chegar no destinatário, a mensagem executa a função
cujo nome termina em `_Implementation`.

- `Reliable`: as funções reliables sempre chegam no destinatário e em ordem de envio. mais pesado.
- `Unreliable`: a chegada não é garantida, nem a ordem. mais leve.

#### `UFUNCTION(Server, Reliable|Unrealiable)`
Manda uma mensagem de um client para o servidor.

- Se chamada no servidor, a implementação é executada no servidor
- Se chamada no client, a implementação é executada no servidor

Essas fuções têm o prefixo `Server` por padrão.

#### `UFUNCTION(Client, Reliable|Unreliable)`
Manda uma mensagem do servidor para o client que é dono do Actor onde foi chamada a função. Não muito comum na prática.

- Se chamada no servidor, a implementação é executada no client dono do Actor
- Se chamada no client, a implementação é executada no mesmo client

Essas fuções têm o prefixo `Client` por padrão.

#### `UFUNCTION(NetMulticast, Reliable|Unreliable)`
Manda uma mensagem do servidor para todos os clients.

- Se chamada no servidor, a implementação é executada no servidor e em todos os clients
- Se chamada no client, a implementação é executada no mesmo client

Essas fuções têm o prefixo `Multicast` por padrão.

Ex:
```cpp
// .h
UFUNCTION(NetMulticast, Reliable)
void MulticastSayHi();

// .cpp
void AMyActor::BeginPlay()
{
	if (GetLocalRole() == ROLE_Authority)
	{
		// apenas o servidor manda a mensagem
		MulticastSayHi();
	}
}

void AMyActor::MulticastSayHi_Implementation()
{
	// executado no servidor e em todos os clients
	if (GEngine != nullptr)
	{
		GEngine->AddOnScreenDebugMessage(
			-1,
			0.0f,
			FColor::Red,
			TEXT("HI!")
		);
	}
}
```

--------------------

## topicos
- recap ponteiros
	- "stale pointer"
		- casos comuns
			- "guardar um ponteiro vindo do actor em um component dentro de seu construtor"
		- quando o IsValid pode falhar
	- vs referencias
		- evitar usar referencias não const
	- TWeakObjPtr<T>
- const
	- const ptr, const data
		- regrinha da espiral da declaração
	- const method
- memória
	- alocadores
	- TArray com alocador geral e inline
	- usar array simples quando possivel
	- TArrayView
- declarações
	- construtores
		- stack, copy, etc
			- `Type MyVar = Type("args");`

