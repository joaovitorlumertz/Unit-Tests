# Testes unitários
O código dos testes é tão importante quanto o código de produção. 

A seguir, veremos como podemos aplicar alguns conceitos que podem nos ajudar no desenvolvimento de testes legíveis e bem estruturados.

# Sumário
1. [Falsos positivos]()
2. [Desalocar propriedades da classe de teste]()
3. [Given - When - Then]()
4. [Fixtures]()
5. [Spies]()
6. [Injeção de dependência]()
7. [Snapshots]()

## 1. Falsos positivos
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
// Esse teste passará ✅, porém não testamos nada nele. 
func test_doSomething() {
    sut.duSomething()
}
```

Para tornar o teste melhor, precisamos verificar se a função está realizando a tarefa esperada:

```swift
func test_doSomething_shouldPresentAlert() {
    // when
    sut.doSomething()

    // then
    XCTAssertTrue(viewSpy.presentAlertCalled)
}
```

## 2. Desalocar propriedades da classe de teste

Estamos acostumados a definir propriedades em nossas classes de testes. Normalmente, as definimos em um método `setUp` para que sejam configuradas corretamente para cada método de teste.

Vamos ver como o `XCTest` executa nossos testes e qual o ciclo de vida das nossas classes de teste e suas propriedades.

Observe o playground a seguir:

```swift
import Foundation
import XCTest

class SomeClass: NSObject {
    
    func foo() {
        print("\n ===== foo from \(self) \n")
    }
    
    func bar() {
        print("\n ===== bar from \(self) \n")
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
Test Suite 'SomeClassTestCase' started at 2023-02-08 14:47:21.483
Test Case '-[__lldb_expr_10.SomeClassTestCase test_bar]' started.

 ===== bar from <__lldb_expr_10.SomeClass: 0x600001648360> 

Test Case '-[__lldb_expr_10.SomeClassTestCase test_bar]' passed (0.010 seconds).
Test Case '-[__lldb_expr_10.SomeClassTestCase test_foo]' started.

 ===== foo from <__lldb_expr_10.SomeClass: 0x6000016483a0> 

Test Case '-[__lldb_expr_10.SomeClassTestCase test_foo]' passed (0.001 seconds).
Test Suite 'SomeClassTestCase' passed at 2023-02-08 14:47:21.496.
	 Executed 2 tests, with 0 failures (0 unexpected) in 0.011 (0.013) seconds

```
Observe que para cada método de teste há uma instância diferente da classe `SomeClass`. Em resumo, cada método de teste cria uma nova instância da classe de teste.

Ao executar os testes, o `XCTest` cria uma lista - um array em memória - com todas as instâncias das subclasses de `XCTestCase`. Ao final da execução de todas as classes de teste, um relatório é gerado, assim como vimos no log a cima.

Sabendo que as instâncias ficam salvas em meméoria, podemos imaginar que as propriedaes de cada `TestCase` também são alocadas.

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

Observe que a classe `SomeClass` não foi desalocada da memória. Em geral, isso não deve ser um problema. Porém, em grandes projetos isso pode ser problemático para devs que não possuem um Mac tão recente. Além disso, pode diminuir o desempenho da execução dos testes em máquinas de CI, que geralmente não possuem um hardware muito avançado e com grande capacidade de memória e processamento.

### Solução

Precisamos desalocar as propriedades da classe de teste.

Para isso precisamos atribuir as propriedades com `nil` depois que cada método de teste for executado. Os métodos `setUp` e `tearDown` podem nos ajudar com isso:

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

## 3. Given - When - Then
Given-When-Then é um estilo de representação de testes. Foi originiado junto ao BDD (Behavior Driven Development) com o intuito de documentar requisitos e testes. De forma simples, significa que o teste será dividido em três partes:

- **Given**: Definimos a configuração do estado inicial de um cenário de teste. Por exemplo, podemos configurar um Stub, definir valores em propriedades da classe em teste (sut) ou chamar alguma função que define o estado inicial para o que irá ser testado.

- **When**: Executamos método do SUT que queremos testar.

- **Then**: Realizamos as verificações necessárias para validar o teste (XCTAssert).

**Antes**
```swift
func test_viewDidAppear_withAuthenticatedUserWithQRCode_shouldCallRepositoryAddQRCode() {
    let cpf = "12345678910"
    stateManagerSpy.isUserLogged = true
    stateManagerSpy.cpf = cpf
    let expectedQRCode = QRCode.dummy(customerCPF: cpf)
    userDefaultsSpy.valueSet = expectedQRCode
    sut.viewDidAppear()
    XCTAssertTrue(repositorySpy.isAddQRCodeCalled)
    XCTAssertEqual(repositorySpy.addQRCodeParam, expectedQRCode)
}
```

**Depois**
```swift
func test_viewDidAppear_withAuthenticatedUserWithQRCode_shouldCallRepositoryAddQRCode() {
    // given
    let cpf = "12345678910"
    stateManagerSpy.isUserLogged = true
    stateManagerSpy.cpf = cpf
    let expectedQRCode = QRCode.dummy(customerCPF: cpf)
    userDefaultsSpy.valueSet = expectedQRCode

    // when
    sut.viewDidAppear()
    
    // then
    XCTAssertTrue(repositorySpy.isAddQRCodeCalled)
    XCTAssertEqual(repositorySpy.addQRCodeParam, expectedQRCode)
}
```

### Helpers
Para diminuir a complexidade e melhorar o reaproveitamento de código em nossos testes, utilizamos funções auxiliares (helpers) que abstraem parte da lógica de nossos casos de teste (makeSUT, givenSomeState, whenFetchData, etc...). Veja um exemplo de como podemos melhorar o teste anterior:

```swift
// MARK: - Given
private func givenAuthenticatedUserWithQRCode() -> QRCode {
    let cpf = "12345678910"
    stateManagerSpy.isUserLogged = true
    stateManagerSpy.cpf = cpf
    let qrCode = QRCode.dummy(customerCPF: cpf)
    userDefaultsSpy.valueSet = qrCode
    return qrCode
}

// MARK: - viewDidAppear
func test_viewDidAppear_withAuthenticatedUserWithQRCode_shouldCallRepositoryAddQRCode() {
     // given
    let expectedQRCode = givenAuthenticatedUserWithQRCode()
        
    // when
    sut.viewDidAppear()
        
    // then
    XCTAssertTrue(repositorySpy.isAddQRCodeCalled)
    XCTAssertEqual(repositorySpy.addQRCodeParam, expectedQRCode)
}
```

Agora veja outro exemplo, neste caso, estamos abstraindo a etapa **When** do caso de teste:

**Antes**
```swift
func test_showFormAboutYouInfo_shouldSetCurrentViewController_withViewControllerOfCorrectType() {
    // given
    let dummy = RequestCardAboutYouViewState()

    // when
    sut.showFormAboutYouInfo(statePersonalInfo: dummy)

    // then
    XCTAssertTrue(sut.currentViewController is RequestCardAboutYouViewController)
}

func test_showFormAboutYouInfo_shouldPushViewControllerOfCorrectType() {
    // given
    let dummy = RequestCardAboutYouViewState()

    // when
    sut.showFormAboutYouInfo(statePersonalInfo: dummy)

    // then
    XCTAssertTrue(navigationControllerSpy.pushViewController is RequestCardAboutYouViewController)
}
```

**Depois**
```swift
// MARK: - When
private func whenShowFormAboutYouInfo() {
    let dummy = RequestCardAboutYouViewState()

    sut.showFormAboutYouInfo(statePersonalInfo: dummy)
}

// MARK: - showFormAboutYouInfo
func test_showFormAboutYouInfo_shouldSetCurrentViewController_withViewControllerOfCorrectType() {
    // when
    whenShowFormAboutYouInfo()

    // then
    XCTAssertTrue(sut.currentViewController is RequestCardAboutYouViewController)
}

func test_showFormAboutYouInfo_shouldPushViewControllerOfCorrectType() {
    // when
    whenShowFormAboutYouInfo()

    // then
    XCTAssertTrue(navigationControllerSpy.pushViewController is RequestCardAboutYouViewController)
}

```

## 4. Fixtures
// TODO

## 5. Spies (enum calledMethods)
// TODO

## 6. Injeção de dependência
// TODO

## 7. Snapshots
// TODO
