# C++ na Unreal

Um guia de sobrevivência para programadores C++ em Unreal.

Lembrando que isso não substitui ler a documentação da Unreal (e às vezes até mesmo o código fonte também),
mas é um bom começo.

# Coisinhas de C++

## `#include "SomeHeaderFile.h"`


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
UPROPERTY(Transient)
UMovementComponent* MovementComponent; // valor vem da blueprint que herda da classe

// .cpp
void AMyActor::BeginPlay()
{
	Super::BeginPlay();

	// propriedade sempre inicializado
	MovementComponent = (UMovementComponent*)FindComponentByClass(UMovementComponent::StaticClass());
}
```

## `UFUNCTION()`
Marca uma função para ser reconhecida pela Unreal.
Também pode ter vários sub-atributos que extendem a semântica da função.

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
		GEngine->AddOnScreenDebugMessage(-1, 0.0f, FColor::Red, TEXT("HI!"));
	}
}
```

--------------------

## topicos
- #include
	- processo de compilação c++
	- minimizar include em .h
		- class T*
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
- static
	- static function
- declarações
	- construtores
		- stack, copy, etc
			- `Type MyVar = Type("args");`

