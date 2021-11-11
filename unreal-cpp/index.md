# C++ na Unreal

Um guia de sobrevivência para programadores C++ em Unreal.

Lembrando que isso não substitui ler a documentação da Unreal (e às vezes até mesmo o código fonte também),
mas é um bom começo.

# Coisinhas de C++

## Tipos básicos

### `int`
Curiosamente, o _standard_ do C/C++ não define os tamanhos de `int`, `short int`, `long int` e nem mesmo `char`.
Para lidar com isso, a Unreal define os tipos `int8`, `int16`, `int32`, `int64` para números inteiros e, similarmente,
`uint8`, `uint16`, `uint32` e `uint64` para números naturais.

Prefira sempre utilizar esses tipos ao inves dos padrões pois isso nos dá garantia de quantos bits realmente estamos usando
principalmente quando se tratar de *armazenar* dados (i.e. um campo de inteiro em uma classe sua, etc) .

### `bool`
Igualmente, C++ não define o tamanho que um `bool` ocupa (pode ser 1 byte, pode ser 4 bytes, pode ser um valor maluco).
Por conta disso que quando fazemos uma `UPROPERTY()` booleana, costumamos declará-la como:

```cpp
UPROPERTY(...)
uint32 bSomeFlag : 1;
```

O `: 1` depois do nome do campo significa que aquele `int` usa apenas `1 bit` em sua representação armazenada (dentro da classe).
Isso é coisa de C/C++ mesmo, e deve permitir a Unreal fazer otimizações quando estiver serializando a classe (tanto pra disco
como pra networking também).

Ainda assim, é comum ver funções retornando `bool` na base de código da Unreal. Nesses casos é tudo bem pois são apenas
valores temporários e não estão sendo armazenados em nenhuma `UPROPERTY()`.

Então a dica é usar `bool` quando for valor temporário e, quando for criar uma `UPROPERTY()`, use o idioma `uint32 bMyBool : 1;`.

### `float`
Igualmente curioso é que o _standard_ também não define o tamanho de `float` e `double` em C/C++.
Porém, como basicamente toda plataforma suportada pela Unreal possui uma unidade de processamento de `float` que
seguem o padrão `IEEE-754`, acaba que sabemos a priori que `float`s são 32 bits e `double`s, 64 bits.

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
Ponteiros podem ser interpretados como números.
Números que representam um endereço de memória que *pode* conter dados úteis.

O *pode* é porque às vezes ponteiros são nulos (`nullptr`) ou pior: apontam para
memória não mais válida (que pode ou não causar _crash_ quando acessada).

Trabalhando com ponteiros é útil pensar em quem é *dono* dos dados apontados.
Isso porque geralmente podemos inferir o _lifetime_ dos dados a partir dessa informação.

Saber o _lifetime_ dos dados apontados é importante pois se um ponteiro vive mais
que os dados apontados, *coisas estranhas* podem acontecer ao tentar acessá-lo.

No caso da Unreal, você pode categorizar ponteiros em três _lifetime_ possíveis:

- _static_: objetos que sempre estão lá (ex: data assets e globais)
- _self_: objetos que têm o mesmo lifetime que o objeto que contêm o ponteiro (ex: actor que tem ponteiro para um componente próprio)
- _unknown_: objetos cujo _lifetime_ não é sabido previamente (ex: actor que tem ponteiro para outro actor que pode ser destruído a qualquer momento)

#### Ex de um ponteiro que vive mais que os dados apontados:
- actor A guarda um ponteiro para actor B
- actor B é destruído por qualquer motivo
- o ponteiro que A guarda ainda contém o endereço de memória onde o actor B estava
- mesmo o ato de chamar `IsValid()` pode crashar o jogo pois estaríamos acessando memória inválida

#### Ex de um ponteiro que vive tanto quanto os dados apontados:
- actor A guarda um ponteiro para um componente seu
- quando o actor A é destruído, seu componente também é
- porém tudo bem pois o ponteiro também não será mais acessado!

Dificilmente um ponteiro que se encontra em um dos dois primeiros casos irá apontar para memória inválida.
Porém o terceiro caso é onde o perigo se encontra e há basicamente duas formas de se proteger:
- saber exatamente o momento que os dados apontados são destruídos e invalidar os ponteiros imediatamente (botando pra `nullptr` por exemplo)
	- pra isso você precisará ter controle absoluto de todos os ponteiros que apontavam para o recurso e saber exatamente quando este é destruído
- usar um `TWeakObjectPtr` e sempre checar se o ponteiro é válido antes de usá-lo
	- o `TWeakObjectPtr` é específico para objetos da Unreal (`UObject`)
	- esses ponteiros têm conhecimendo do garbage collector da Unreal e por isso conseguem saber quando o ponteiro não aponta mais para dados válidos

### Ponteiros vs Referências
Outro aspecto que C++ possui é referências. Estas são extramemente semelhante a ponteiros com apenas algumas diferenças:
- referências *sempre* são inicializadas durante a declaração
	- uma classe com um campo referência precia que todos os seus contrutores inicializem as referencias *antes* do corpo do construtor
		- `MyActor::MyActor() : MyRef(InitializationExpression) { /* constructor body */ }`
- é invisível em uma chamada de função se ela está recebendo os argumentos por valor ou por referencia
	- `int Result = DoCalculation(MyMatrix); // por valor ou por referencia?? vai saber!!`

Principalmente por conta do segundo ponto, eu recomendo *muito* que caso forem criar uma função que recebe
uma refência, que seja uma referência para dados que não mudam (`const`). Porque desse modo, meio que não
importa a resposta de se o paraâmetro é por valor ou por referencia já que o argumento passado garantidamente*
não terá seu valor alterado pela função.

[*] Mentira, não é garantido. Isso porque a função pode ter sido implementada *na malandrage* por fazer
uns casts que removem o `const` e ainda assim alterar os valores sem você saber.
Mas aí a gente dá um voto de confiança que não tem ninguém tentando sabotar o seu trabalho, né :')

Por fim, ponteiros são bem mais comuns em bases de código de projetos Unreal.
Então sugiro seguir com os padrões que a engine usa.

## `const`
Marcar algo como `const` significa que aquele valor não vai mudar (você promete pro compilador que não vai mexer com
aquele valor e ele vai acreditar em você!).

Const é bem útil em dois casos:
- ponteiros que apenas irão ler os dados apontados
- métodos que não alteram os campos da classe

### `const Thing*`
`const UActorComponent* MyComponent` ou `UActorComponent const* MyComponent` significam que `MyComponent` é um ponteiro
que apenas pode ler o `UActorComponent` para o qual aponta.

É útil saber que um ponteiro é `const` pois:
- em parâmetros de função significa que você sabe que aquela função *não* irá alterar os argumentos que você passa pra ela
- em retornos de métodos significa que você pode emprestar um dado interno da sua classe com a certeza que ele não será alterado
- em classes significa que você sabe que uma instância *jamais* irá alterar os dados apontados

### Método `const`
Método marcado como `const` nada mais é que um método cujo `this` é `const`. Ou seja, você não pode alterar
os dados da classe de dentro de um método `const`.

Isso é útil pois alguém chamando um método de sua classe sabe com certeza apenas olhando a assinatura do método
que ele não irá alterar o estado da classe.

Ex:
```cpp
// .h
UCLASS()
class AMyActor : AActor
{
	GENERATED_BODY()
public:
	bool IsAlive() const;
private:
	uint32 Health;
}

// .cpp
// OK
bool AMyActor::IsAlive() const
{
	return Health > 0;
}

// ERROR
bool AMyActor::IsAlive() const
{
	Health *= 2; // não pode alterar valor
	return Health > 0;
}
```

### sempre que possível, use `const`
Quando lendo código, `const` diminui a quantidade de coisas a serem consideradas e, portanto, facilitam sua compreensão.

```cpp
uint32* HealthPtr = &MyActor->Health;
UE_LOG(LogTemp, Display, TEXT("health: %d"), *HealthPtr);
// health = 10

MyActor->DoThing();
MyActor->DoOtherThing();

UE_LOG(LogTemp, Display, TEXT("health: %d"), *HealthPtr);
// será que health ainda é 10?
// se `DoThing()` e `DoOtherThing()` são `const`,
// sabemos com certeza que é 10!
```

## _Pitfalls_ pra ficar de olho

### Construtor não default na _stack_
Se você quer criar um valor na _stack_ (sem alocação dinâmica) chamando um construtor que recebe parâmetros,
cuidado para não acabar chamando o construtor de cópia sem querer:

```cpp
// aqui chamamos o construtor que recebe 3 floats
// e em seguida chamamos o construtor de cópia
// para passar esse valor temporário construído
// para a nossa variável `Position`
FVector Position = FVector(0.0f, 0.0f, 1.0f);
```

```cpp
// aqui simplesmente inicializamos nossa
// variável `Position` diretamente chamando
// o construtor correto nela e nada a mais
FVector Position(0.0f, 0.0f, 1.0f);
```

### Divisão de `int`s
`10` em C++ é *sempre* um número inteiro! Por tanto, quando fazemos `10 / 3`, o resultado é sempre `3`
(nenhuma casa decimal). Isso mesmo que a variável que está recebendo a expressão seja `float`!

Se quiser `float`s literais, sempre ponha o `.0f` no final!

Ex:
```cpp
float r1 = 10 / 3; // r1 = 3.0f
float r2 = 10.0f / 3.0f // r2 = 3.333333f
```

# Coisinhas de Unreal

## `UPROPERTY()`
Marca um campo como uma propriedade que a Unreal entende.
Pode ter vários sub-atributos que extendem a semântica da propriedade.

```cpp
UPROPERTY()
UPrimitiveComponent* MyComponent;
```

Importante para campos que são ponteiros para `UObject` que ajuda o garbage collector da Unreal saber
quando ainda tem referencia para um determinado objeto.

### `UPROPERTY(Transient)`
Uma propriedade `Transient` significa que ela não é serializada. Isso é: ela não tem seu valor vindo do disco
(ou do editor por exemplo). Ao invés disso a Unreal assume que você sempre irá iniciar o seu valor manualmente.

Ex:
```cpp
// .h
// Nesse caso hipotético, o componente é definido na
// Blueprint que herda de nossa classe. No begin play,
// nós enfim procuramos pelo componente
UPROPERTY(Transient)
UMovementComponent* MovementComponent;

// .cpp
void AMyActor::BeginPlay()
{
	Super::BeginPlay();

	// A propriedade é *sempre* inicializada
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

## Containers e Memória
A Unreal possui um arcabouço de containers template prontos para serem usados.

### `TArray<ElementType, AllocatorType>`
Comparável ao `List<T>` do C#, esse container é suficiente para 95% dos nossos casos de uso.

Se usado como `TArray<ElementType>`, o allocador escolhido é o `FDefaultAllocator` (que herda de `TSizedHeapAllocator`)
que simplesmente irá alocar os elementos do array na heap. Porém existem outros tipos de alocadores que podem trazer
ganhos de desempenho se você conhece os dados que está tratando.

Porém sabia que, em geral, as APIs da Unreal que tratam com `TArray` esperam que você use o `TArray` com
o alocador padrão. Então infelizmente não é sempre que você consegue se livrar de alocações dinâmicas.

### `TInlineAllocator<uint32 N>`
Esse vai armazenar os primeiros `N` elementos junto do container (na stack se o container foi declarado lá).
Quando se sabe que na maioria dos casos, o número de elementos nunca passa de um determinado valor razoável,
podemos usar esse alocador no lugar do padrão.

O interessante é que, se por acaso o número de elementos passar de `N`, ele passa a então a alocar da heap
automaticamente então o mesmo código consegue tratar alocações grandes.

### `TFixedAllocator<uint32 N>`
Exatamente a mesma coisa que o `TInlineAllocator` porém a alocação falha caso tente alocar mais de `N` elementos.
Apenas use esse alocador quando você tem certeza que o número de elementos nunca passará de `N`.

Nesse ponto, considere também usar um array padrão de C++ no lugar.

### `FHeapAllocator`
O alocador padrão que irá sempre pegar memória da heap. Sempre irá funcionar, porém sempre será o mais pesado.

### `TArrayView<ElementType>`
Quando você quer passar uma lista de elementos para uma função que somente irá lê-los (ao inves de remover ou adicionar
elementos à coleção), você pode usar o `TArrayView` no lugar. Essa interface permite que a função use a variável
como se fosse um array _readonly_ e, ao mesmo tempo, é compatível com `TArray`s de qualquer alocador e até mesmo
arrays padrão de C++.

Outro motivo para usar esse tipo nas funções é que fica bem claro pela assinatura que a função apneas irá
_ler_ os elementos e nunca adicionar ou remover. Parecido com os motivos de usar `const`.

Porém infelizmente não é possível usar esse tipo em funções que são chamáveis por Blueprint ou RPCs (nesses
casos é sempre `TArray` com o alocador padrão).

