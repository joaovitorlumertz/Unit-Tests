## Sumário
1. [Testes unitários](#1-testes-unitários)
2. [Falsos positivos](#2-falsos-positivos)
3. [Given-When-Then](#3-given---when---then)
4. [Testes menores e eficientes](#4-testes-menores-e-eficientes)
5. [Desalocar propriedades da classe de teste](#5-desalocar-propriedades-da-classe-de-teste)
6. [Injeção de dependência](#6-injeção-de-dependência)
7. [Spies](#7-spies)
8. [Fixture](#8-fixture)
9. [Snapshots](#9-snapshots)

## 1. Testes unitários
Os testes unitários servem para testar uma unidade do código.

🧐 O que é uma unidade?
- Uma unidade é uma função ou método.

### Testes unitários x Teste de integração
É muito comum, confundir testes unitários com testes de integração. Vamos ver a diferença entre eles: 

- Um teste de **integração** testa **mútiplas unidades** do código para garantir que elas funcionem juntas como o esperado. Os testes de integração são realizados após os testes de unidade e podem pegar problemas que não foram detectados pelos testes de unidade, como problemas em fluxos de dados e comunicação entre componentes.

- Um teste de **unidade** testa **unidades individuais** do código, como funções ou métodos **isolados** do restante do sistema. Eles são muito utilizados durante o desenvolvimento para encontrar bugs mais cedo. A meta dos testes de unidade é garantir que cada unidade do código funcione como esperado.

### Vantagens dos testes unitários
- É o tipo de teste mais simples de implementar;
- Refatoração do código com segurança;
- Encontrar bugs cedo;
- Dimiuir o acoplamento do código.

### O que testar?
- Funções públicas (internal, pulic, open);
- Comunicação entre objetos;
- Regras de negócio.

## 2. Falsos positivos
Para que um teste seja efeitivo, devemos definir as asserções necessárias para que o teste passe. Escrever um teste sem asserções não fará com que ele falhe. O XCode indicará que o trecho de código foi coberto por testes, porém nada foi testado de fato.

Veja o exemplo a seguir:

```swift
class SomePresenter {
    weak var view: SomeViewProtocol?
    
    func doSomething() {
        view?.presenterAlert()
    }
}
```


```swift
// ❌ Esse teste passará, porém não testamos nada nele. 
func test_doSomething() {
    sut.duSomething()
}
```

Para tornar o teste melhor, precisamos verificar se a função está realizando a tarefa esperada:

```swift
// ✅ Agora estamos validando a função de fato
func test_doSomething_shouldPresentAlert() {
    // when
    sut.doSomething()

    // then
    XCTAssertTrue(viewSpy.presentAlertCalled)
}
```

[back to top](#sumário)


## 3. Given - When - Then
Given-When-Then é um estilo de representação de testes. Foi originiado junto ao BDD (Behavior Driven Development) com o intuito de documentar requisitos e testes. De forma simples, significa que o teste será dividido em três partes:

- **Given**: Definimos a configuração do estado inicial de um cenário de teste. Por exemplo, podemos configurar um Stub, definir valores em propriedades da classe em teste (sut) ou chamar alguma função que define o estado inicial para o que irá ser testado.

- **When**: Executamos o método do SUT que queremos testar.

- **Then**: Realizamos as verificações necessárias para validar o teste (`XCTAssert`).

**Antes**
```swift

func fetchUserData(userId: Int) {
    requester.request(.fetchUserData(userId)) { [weak self] result in
        switch result {
            case .success(let data):
                self?.output?.fetchUserDataSucceeded(data)
            case .error(let error):
                self?.output?.fetchUserDataFailed(error)
        }
    }
}

func test_fetchUserData_withSuccessRequest_shouldCallOutputSuccessMethod() {
    let userData = userData.dummy()
    requesterStub.setReturn(userData)
    sut.fetchUserData(userId: 1)
    XCTAssertTrue(outputSpy.fetchUserDataSucceededCalled)
}
```

**Depois**
```swift
func test_fetchUserData_withSuccessRequest_shouldCallOutputSuccessMethod() {
    // given
    let userData = userData.dummy()
    requesterStub.setReturn(userData)

    // when
    sut.fetchUserData(userId: 1)
    
    // then
    XCTAssertTrue(outputSpy.fetchUserDataSucceededCalled)
}
```
[back to top](#sumário)

### Helpers
Para diminuir a complexidade e melhorar o reaproveitamento de código em nossos testes, utilizamos funções auxiliares (helpers) que abstraem parte da lógica de nossos casos de teste (`makeSUT`, `givenSomeState`, `whenFetchData`, etc...). 

Veja um exemplo de como podemos melhorar o teste anterior:

```swift
// MARK: - Given
private func givenFetchUserDataSucceeded() {
    let userData = userData.dummy()
    requesterStub.setReturn(userData)
}

// MARK: - fetch user data
func test_fetchUserData_withSuccessRequest_shouldCallOutputSuccessMethod() {
    // given
    givenFetchUserDataSucceeded()
        
    // when
    sut.fetchUserData(userId: 1)
        
    // then
    XCTAssertTrue(outputSpy.fetchUserDataSucceededCalled)
}
```

Agora veja outro exemplo, neste caso, estamos abstraindo a etapa **When** do caso de teste:

**Antes**
```swift
func test_fetchUserData_withSuccessRequest_shouldCallOutputSuccessMethod() {
    // given
    givenFetchUserDataSucceeded()
        
    // when
    sut.fetchUserData(userId: 1)
        
    // then
    XCTAssertTrue(outputSpy.fetchUserDataSucceededCalled)
}
```

**Depois**
```swift
// MARK: - When
private func whenFetchUserData() {
    sut.fetchUserData(userId: 1)
}

// MARK: - showFormAboutYouInfo
func test_fetchUserData_withSuccessRequest_shouldCallOutputSuccessMethod() {
    // given
    givenFetchUserDataSucceeded()

    // when
    whenFetchUserData()

    // then
    XCTAssertTrue(outputSpy.fetchUserDataSucceededCalled)
}

```
[back to top](#sumário)

## 4. Testes menores e eficientes
Uma prática muito comum no TDD é utilizar uma única asserção por teste. Com métodos de teste bem nomeados, quando o teste falhar, você saberá exatamente onde está o problema porque não há ambiguidade entre várias condições. 

Um teste com muitas afirmações torna difícil fornecer um nome ou descrição significativa. Contudo, isso não significa que todos os testes devam ter apenas uma afirmação. Conceitualmente, afirmar propriedades de um mesmo objeto pode-se considerar um único assert.

🚩 Os testes a seguir realizam mais de uma validação:
```swift
func loginSucceeded(_ data: User) {
    if user.admin {
        coordinator.navigateToAdminHomePage()
    } else {
        coordinator.navigateToHomePage()
    }

    stateManager.user = user
    notificationCenter.post(name: Notification.Name(rawValue: "login_succeeded"), 
                            object: self, 
                            userInfo: ["user": user])
}
```

```swift
func test_loginSucceeded_whenIsAdmin_shouldNavigateToAdminHomePage_andSetUserOnState_andNotifyLogin() {
    // given
    let expectedUser: User = .dummyAdmin()
    var observedNotifiedUser: User? = nil

    let handler: (Notification) -> Bool { notification in 
        guard let user = notification.userInfo?["user"] as? User else {
            return false
        }

        observedNotifiedUser = user

        return true
    }

    expectation(forNotification: Notification.Name(rawValue: "login_succeeded"),
                object: nil,
                handler: handler)

    // when
    sut.loginSucceeded(expectedUser)

    // then
    XCTAssertTrue(coordinatorSpy.navigateToAdminHomePageCalled)
    XCTAssertEqual(stateManagerSpy.user, expectedUser)
    XCTAssertEqual(observedNotifiedUser, expectedUser)
    waitForExpectations(timeout: 1, handler: nil)
}
```

```swift
func test_loginSucceeded_shouldNavigateToHomePage_andSetUserOnState_andNotifyLogin() {
    // given
    let expectedUser: User = .dummy()
    var observedNotifiedUser: User? = nil

    let handler: (Notification) -> Bool { notification in 
        guard let user = notification.userInfo?["user"] as? User else {
            return false
        }

        observedNotifiedUser = user

        return true
    }

    expectation(forNotification: Notification.Name(rawValue: "login_succeeded"),
                object: nil,
                handler: handler)

    // when
    sut.loginSucceeded(expectedUser)

    // then
    XCTAssertTrue(coordinatorSpy.navigateToHomePageCalled) // Esta é a condição que mudou
    XCTAssertEqual(stateManagerSpy.user, expectedUser)
    XCTAssertEqual(observedNotifiedUser, expectedUser)
    waitForExpectations(timeout: 4, handler: nil)
}
```

✅ Podemos criar um teste para cada unidade da nossa função:

```swift
func test_loginSucceeded_whenIsAdmin_shouldNavigateToAdminHomePage() {
    // when
    sut.loginSucceeded(.dummyAdmin())

    // then
    XCTAssertTrue(coordinatorSpy.navigateToAdminHomePageCalled)
}

func test_loginSucceeded_shouldNavigateToHomePage() {
    // when
    sut.loginSucceeded(.dummy())

    // then
    XCTAssertTrue(coordinatorSpy.navigateToHomePageCalled)
}

func test_loginSucceeded_shouldSetUserOnState() {
    // given
    let expectedUser: User = .dummy()

    // when
    sut.loginSucceeded(expectedUser)

    // then
    XCTAssertEqual(stateManagerSpy.user, expectedUser)
}

func test_loginSucceeded_shouldNotifyLogin() {
    // given
    let expectedUser: User = .dummy()
    var observedNotifiedUser: User? = nil

    let handler: (Notification) -> Bool { notification in 
        guard let user = notification.userInfo?["user"] as? User else {
            return false
        }

        observedNotifiedUser = user

        return true
    }

    expectation(forNotification: Notification.Name(rawValue: "login_succeeded"),
                object: nil,
                handler: handler)

    // when
    sut.loginSucceeded(expectedUser)

    // then
    XCTAssertEqual(observedNotifiedUser, expectedUser)
    waitForExpectations(timeout: 4, handler: nil)
}

```


[back to top](#sumário)

## 5. Desalocar propriedades da classe de teste

Estamos acostumados a definir propriedades em nossas classes de testes. Normalmente, as definimos em um método `setUp` para que sejam configuradas corretamente para cada método de teste.

Vamos ver como o `XCTest` executa nossos testes e qual o ciclo de vida das nossas classes de teste e suas propriedades.

Observe o playground a seguir:

```swift
import Foundation
import XCTest

class SomeClass: NSObject {
    
    func foo() {
        print(">>>>> foo from \(self)")
    }
    
    func bar() {
        print(">>>>> bar from \(self)")
    }
    
}


class SomeClassTestCase: XCTestCase {
    
    let sut = SomeClass()
    
    func test_foo() {
        sut.foo()
    }
    
    func test_bar() {
        sut.bar()
    }
    
}

SomeClassTestCase.defaultTestSuite.run()

```
O log gerado pelo caso de teste foi:

```
Test Suite 'SomeClassTestCase' started at 2023-02-09 09:04:52.950
Test Case '-[__lldb_expr_1.SomeClassTestCase test_bar]' started.
>>>>> bar from <__lldb_expr_1.SomeClass: 0x60000362c050>
Test Case '-[__lldb_expr_1.SomeClassTestCase test_bar]' passed (0.052 seconds).
Test Case '-[__lldb_expr_1.SomeClassTestCase test_foo]' started.
>>>>> foo from <__lldb_expr_1.SomeClass: 0x60000362c080>
Test Case '-[__lldb_expr_1.SomeClassTestCase test_foo]' passed (0.002 seconds).
Test Suite 'SomeClassTestCase' passed at 2023-02-09 09:04:53.061.
	 Executed 2 tests, with 0 failures (0 unexpected) in 0.054 (0.111) seconds


```

Observe que para cada método de teste há uma instância diferente da classe `SomeClass`. Em resumo, cada método de teste cria uma nova instância da classe de teste.

Ao executar os testes, o `XCTest` cria uma lista - um array em memória - com todas as instâncias das subclasses de `XCTestCase`. Ao final da execução de todas as classes de teste, um relatório é gerado, assim como vimos no log a cima.

Todas as instâncias das classes de testes ficam salvas em memória até o término da execução, por consequência, cada propriedade dessas classes também ficará alocada na memória.

Vamos observar isso acrescentado um `init` e `deinit` na classe `SomeClass`:

```swift
class SomeClass: NSObject {
    
    override init() {
        super.init()
        print("\n ===== init from \(self) \n")
    }
    
    deinit {
        print("\n ===== deinit from \(self) \n")
    }
    
    func foo() {
        print("\n ===== foo from \(self) \n")
    }
    
    func bar() {
        print("\n ===== bar from \(self) \n")
    }
    
}

```

Ao rodar o teste novamente obtemos o seguinte log:

```
>>>>> init from <__lldb_expr_3.SomeClass: 0x6000028f8210>
>>>>> init from <__lldb_expr_3.SomeClass: 0x6000028e81d0>
Test Suite 'SomeClassTestCase' started at 2023-02-08 21:48:12.316
Test Case '-[__lldb_expr_3.SomeClassTestCase test_bar]' started.
>>>>> bar from <__lldb_expr_3.SomeClass: 0x6000028f8210>
Test Case '-[__lldb_expr_3.SomeClassTestCase test_bar]' passed (0.022 seconds).
Test Case '-[__lldb_expr_3.SomeClassTestCase test_foo]' started.
>>>>> foo from <__lldb_expr_3.SomeClass: 0x6000028e81d0>
Test Case '-[__lldb_expr_3.SomeClassTestCase test_foo]' passed (0.001 seconds).
Test Suite 'SomeClassTestCase' passed at 2023-02-08 21:48:12.342.
	 Executed 2 tests, with 0 failures (0 unexpected) in 0.023 (0.026) seconds

```

Observe que a classe `SomeClass` não foi desalocada da memória. Em geral, isso não deve ser um problema. Porém, em grandes projetos isso pode ser problemático, visto que há milhares de classes de teste. Por consequência, pode diminuir o desempenho da execução dos testes em máquinas de CI, que geralmente não possuem um hardware muito avançado e com grande capacidade de memória e processamento.

### Solução

Precisamos desalocar as propriedades da classe de teste.

Para isso precisamos atribuir as propriedades com `nil` depois que cada método de teste for executado. Os métodos `setUp` e `tearDown` podem nos ajudar com isso:

```swift
final class SomeClassTestCase: XCTestCase {

    var sut: SomeClass!
    
    override func setUp() {
        super.setUp()
        sut = .init()
    }
    
    override func tearDown() {
        sut = nil
        super.tearDown()
    }
    
    func test_foo() {
        sut.foo()
    }
    
    func test_bar() {
        sut.bar()
    }

}
```

```
Test Suite 'SomeClassTestCase' started at 2023-02-08 21:51:16.400
Test Case '-[__lldb_expr_5.SomeClassTestCase test_bar]' started.
>>>>> init from <__lldb_expr_5.SomeClass: 0x6000001a4220>
>>>>> bar from <__lldb_expr_5.SomeClass: 0x6000001a4220>
>>>>> deinit from <__lldb_expr_5.SomeClass: 0x6000001a4220>
Test Case '-[__lldb_expr_5.SomeClassTestCase test_bar]' passed (0.029 seconds).
Test Case '-[__lldb_expr_5.SomeClassTestCase test_foo]' started.
>>>>> init from <__lldb_expr_5.SomeClass: 0x6000001a8420>
>>>>> foo from <__lldb_expr_5.SomeClass: 0x6000001a8420>
>>>>> deinit from <__lldb_expr_5.SomeClass: 0x6000001a8420>
Test Case '-[__lldb_expr_5.SomeClassTestCase test_foo]' passed (0.020 seconds).
Test Suite 'SomeClassTestCase' passed at 2023-02-08 21:51:16.451.
	 Executed 2 tests, with 0 failures (0 unexpected) in 0.049 (0.051) seconds

```

Ao término da execução de cada teste a propriedade é desalocada, com isso diminuimos o consumo de memória.

[back to top](#sumário)

## 6. Injeção de dependência
A classe e suas funções não devem criar os objetos. Uma boa prática consiste em injetar as dependências, de modo que nossas classes não fiquem acopladas com implementações concretas de outros objetos do sistema.

Uma alternativa para injetar as dependências é passá-las no construtor de nossas classes (`init`).

```swift
class NetworkClient {

}

class HomeService {    
    init(network: NetworkClient) {            

    }   
    
    func fetchHomeData(completion: @escaping (HomeData) -> Void)) {

    }
}

struct HomeData {

}

protocol HomeViewModelDelegate {
    func didFetchHomeData(_ data: HomeData)    
}
    
struct HomeViewModel {   
    
    weak var delegate: HomeViewModelDelegate?        
    
    func fetchHomeData() {        
        let network = NetworkClient()        
        let service = HomeService(network: network)                
        
        service.fetchHomeData(completion: { homeData in            
            guard let homeData = homeData else {                
                return            
            }            
            
            delegate?.didFetchHomeData(homeData)        
        })    
    }

}
```

Poderia ficar assim:

```swift
struct HomeViewModel {    

    weak var delegate: HomeViewModelDelegate?        
    private let service: HomeService
    
    init(service: HomeService) {
        self.service = service
    }        
    
    func fetchHomeData() {        
        service.fetchHomeData(completion: { homeData in            
            guard let homeData = homeData else {
                return            
            }            
            
            delegate?.didFetchHomeData(homeData)        
        })    
    }

}
```

Ainda podemos aplicar o conceito de inversão de dependência, onde paramos de depender de tipos concretos e passamos a depender de interfaces (protocolos). Com isso, podemos injetar qualquer objeto que conforme com o protocolo, até mesmo dublês de teste.

```swift

protocol HomeServiceProtocol {
    func fetchHomeData(completion: @escaping (HomeData) -> Void))
}

class HomeService: HomeServiceProtocol {    
    
    init(network: NetworkClient) {

    }
    
    func fetchHomeData(completion: @escaping (HomeData) -> Void)) {

    }

}

struct HomeViewModelFinal {    
    
    weak var delegate: HomeViewModelDelegate?
    private let service: HomeServiceProtocol        
    
    init(service: HomeServiceProtocol) { 
        self.service = service
    }        
    
    func fetchHomeData() {
        service.fetchHomeData(completion: { homeData in            
            guard let homeData = homeData else {
                return            
            }            
            
            delegate?.didFetchHomeData(homeData)        
        })    
    }

}
```

Agora a classe está recebendo o serviço via injeção de dependência, através do `init`. Ela não sabe nada sobre a implementação concreta do serviço, sabe apenas que o serviço deve conformar com o protocolo. Dessa forma, qualquer objeto que implemente o protocolo pode ser injetado na classe. 

**Exemplo de teste**:

Criaríamos um `HomeServiceProtocolSpy` que herdaria de `HomeServiceProtocol`, assim poderíamos observar se os métodos estão sendo chamados, já que nos testes, o Spy vai fazer parte da inicialização do nosso `sut`.

```swift
class HomeServiceProtocolSpy: HomeServiceProtocol {    
    
    enum Method: Equatable {        
        case fetchHomeData()
    }        
    
    private(set) var calledMethods: [Method] = []    
    
    func fetchHomeData(completion: @escaping (HomeData) -> Void)) {
        calledMethods.append(.fetchHomeData)    
    }

}

class HomeViewModelTestCase: XCTestCase {
    
    var sut: HomeViewModelFinal!    
    var serviceSpy: HomeServiceProtocolSpy!        
    
    override func setUp() {        
        super.setUp()
        serviceSpy = HomeServiceProtocolSpy()                
        sut = HomeViewModelFinal(service: serviceSpy)    
    }        
    
    override func tearDown() {
        serviceSpy = nil
        sut = nil    
        super.tearDown()
    }
    
    func test_givenFetchHomeData_shouldFetchData() {
        sut.fetchHomeData()                
        
        XCTAssertEqual(serviceSpy.calledMethods, [.fetchHomeData])    
    }
    
}
```

Perceba que, nesse teste, garantimos que a função `fetchHomeData` é chamada uma única vez. Caso alguém altere o código e duplique a chamada da função, o teste falhará. Por último e não menos imporntante, caso exista outros métodos dentro da classe de serviço, garantimos que eles não estão sendo chamados onde não devem.


## 7. Spies

Objetos “espiões” servem para saber se métodos foram chamados ou não. Os Spies também podem ser usados como Stubs, que servem para controlar o resultado a ser retornado.

```swift
class AddressListAnalyticsInteractorSpy: AddressListAnalyticsInteractorProtocol {
    
    var isTrackScreenViewCalled = false
    var isTrackAddressDeletionCalled = false
    var isTrackAddressHighlightCalled = false    
    
    func trackScreenView() {
        isTrackScreenViewCalled = true
    }
    
    func trackAddressDeletion() {
        isTrackAddressDeletionCalled = true
    }
    
    func trackAddressHighlight(_ address: Address) {
        isTrackAddressHighlightCalled = true
    }

}
```

O jeito mais comum é criar variáveis `Bool` de cada método como visto a cima. Porém, desse jeito, caso alguma `func` seja chamada mais de uma vez, nunca saberemos, a não ser que criemos outra variável do tipo `count`, o que começa a tornar o código maçante. Visto isso, existe uma outra solução, o `calledMethods`. Utilizando essa nova abordagem, o `Spy` ficaria assim:

```swift
class AddressListAnalyticsInteractorSpy: AddressListAnalyticsInteractorProtocol {
    
    enum Method: Equatable {
        case trackScreenView()        
        case trackAddressDeletion()        
        case trackAddressHighlight(Address)
    }    
    
    private(set) var calledMethods: [Method] = []    
    
    func trackScreenView() {        
        calledMethods(.trackScreenView)    
    }    
    
    func trackAddressDeletion() {      
        calledMethods(.trackAddressDeletion)    
    }    
    
    func trackAddressHighlight(_ address: Address) {        
        calledMethods(.trackAddressHighlight(address))    
    }
    
}
```

E verificar se os métodos estão sendo chamados:

```swift
func test_didChangeHighlightedAddress_shouldCallAnalyticsTrackAddressHighlight() {
    let addressModel: Address = .fixture()
    
    sut.didChangeHighlightedAddress(addressModel)                
    
    XCTAssertEqual(analyticsInteractorSpy.calledMethods, [.trackAddressHighlight(addressModel)])
}
```

## 8. Fixture
Mesmo conceito dos `getDummy` já existentes. Serve para criar um mock do modelo desejado, e alterar somente o que precisar, quando precisar.

```swift
extension Address {    

    static let fixture(        
        id = 0,        
        name = "Minha casa",        
        type = "Casa",        
        recipient = "",        
        number = "4",        
        zipCode = "00000-000",        
        referencePoint = "N/A",        
        street = "Rua dos Alfeneiros",        
        neighborhood = "Santo Amaro",        
        city = "São Paulo",        
        state = "SP",        
        isDefaultAddress = true,        
        isValid = true    
    ) -> Address {       
        .init(            
            id: id,            
            name: name,            
            type: type,            
            recipient: recipient,           
            number: number,            
            zipCode: zipCode,            
            referencePoint: referencePoint,            
            street: street,            
            neighborhood: neighbourhood,            
            city: city,            
            state: state,            
            isDefaultAddress: isDefaultAddress,            
            isValid: isValid        
        )    
    }
}
```

Usabilidade:

```swift
func test_didChangeHighlightedAddress_shouldCallAnalyticsTrackAddressHighlight() {        
    let addressModel: Address = .fixture()        
    
    sut.didChangeHighlightedAddress(addressModel)                
    
    XCTAssertEqual(analyticsInteractorSpy.calledMethods, [.trackAddressHighlight(addressModel)])
}
```


## 9. Snapshots

Biblioteca: [swift-snapshot-testing](https://github.com/pointfreeco/swift-snapshot-testing)

Os testes de snapshot auxiliam na validação da interface, podemos verificar se a UI está seguindo o que foi planejado pelo nosso designer e também garantir que futuras mudanças no código não irão "quebrar" o layout existente.

Com o teste de snapshot podemos testar desde pequenos elementos de UI como botões, views, stack views até uma navigation controller ou view controller.

Os Snapshots devem ser gravados sempre com base em um mesmo device, por exemplo, o iPhone 8 Plus. Portanto, utilizar um _wrapper_ que parametriza o snapshot com um padrão comum é bastante útil. Como podemos ver na função `assertSnapshot`:


Para usar o _wrapper_:

```swift
import XCTest
import SnapshoTesting

class SomeViewTestCase: XCTestCase {

    var sut: SomeView!

    override func setUp() {
        super.setUp()
        sut = SomeView()
    }

    override func tearDown() {
        sut = nil
        super.tearDown()
    }

    func test_view_withDefaultConfiguration_shouldMatchSnapshot() {
        sut.frame = CGRect(x: 0, y: 0, width: 375, height: 682)
        assertSnapshot(matching: sut, as: .image)
    }
}
```

A propriedade `isRecording` pode ser utilizada para gerar o snapshot novamente. 
- Quando isRecording é `true`, todo snapshot é gerado novamente. 
- Se atente para definir a propriedade `isRecording` como `false` após validar que o snapshot atende ao layout desejado.

Em alguns casos, nossas views utilizam imagens do servidor, isso pode ser problemátio em testes de snapshot, pois as imagens levam um certo tempo para serem "baixadas", ademais não devemos realizar requisições a serviços externos em nossos testes. Considere utilizar mocks que não possuam URLs reais para as imagens.