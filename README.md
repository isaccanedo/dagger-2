## Introdução ao Dagger 2

# 1. Introdução
Neste tutorial, daremos uma olhada em Dagger 2 - uma estrutura de injeção de dependência rápida e leve.

A estrutura está disponível para Java e Android, mas o alto desempenho derivado da injeção em tempo de compilação o torna uma solução líder para o último.

# 2. Injeção de Dependência
Como um lembrete, a injeção de dependência é uma aplicação concreta do princípio mais genérico de inversão de controle, no qual o fluxo do programa é controlado pelo próprio programa.

É implementado por meio de um componente externo que fornece instâncias de objetos (ou dependências) necessárias para outros objetos.

E diferentes frameworks implementam injeção de dependência de maneiras diferentes. Em particular, uma das mais notáveis ​​dessas diferenças é se a injeção ocorre em tempo de execução ou em tempo de compilação.

A DI em tempo de execução geralmente é baseada na reflexão, que é mais simples de usar, mas mais lenta em tempo de execução. Um exemplo de estrutura de DI em tempo de execução é o Spring.

A DI em tempo de compilação, por outro lado, é baseada na geração de código. Isso significa que todas as operações pesadas são realizadas durante a compilação. DI em tempo de compilação adiciona complexidade, mas geralmente tem desempenho mais rápido.

A Adaga 2 se enquadra nesta categoria.

# 3. Configuração Maven/Gradle
Para usar o Dagger em um projeto, precisaremos adicionar a dependência do dagger ao nosso pom.xml:

```
<dependency>
    <groupId>com.google.dagger</groupId>
    <artifactId>dagger</artifactId>
    <version>2.16</version>
</dependency>
```

Além disso, também precisaremos incluir o compilador Dagger usado para converter nossas classes anotadas no código usado para as injeções:

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.6.1</version>
    <configuration>
         <annotationProcessorPaths>
              <path>
                  <groupId>com.google.dagger</groupId>
                  <artifactId>dagger-compiler</artifactId>
                  <version>2.16</version>
              </path>
         </annotationProcessorPaths>
    </configuration>
</plugin>
```

With this configuration, Maven will output the generated code into target/generated-sources/annotations.

For this reason, we likely need to further configure our IDE if we want to use any of its code completion features. Some IDEs have direct support for annotation processors while others may need us to add this directory to the build path.

Alternatively, if we're using Android with Gradle, we can include both dependencies:

```
compile 'com.google.dagger:dagger:2.16'
annotationProcessor 'com.google.dagger:dagger-compiler:2.16'
```

Agora que temos Dagger em nosso projeto, vamos criar um aplicativo de amostra para ver como funciona.

# 4. Implementação
Para nosso exemplo, tentaremos construir um carro injetando seus componentes.

Agora, Dagger usa as anotações JSR-330 padrão em muitos lugares, sendo um @Inject.

Podemos adicionar as anotações aos campos ou ao construtor. Mas, uma vez que Dagger não suporta injeção em campos privados, iremos para injeção de construtor para preservar o encapsulamento:

```
public class Car {

    private Engine engine;
    private Brand brand;

    @Inject
    public Car(Engine engine, Brand brand) {
        this.engine = engine;
        this.brand = brand;
    }

    // getters and setters

}
```

A seguir, implementaremos o código para realizar a injeção. Mais especificamente, criaremos:

- um módulo, que é uma classe que fornece ou constrói as dependências dos objetos, e
- um componente, que é uma interface usada para gerar o injetor

Projetos complexos podem conter vários módulos e componentes, mas como estamos lidando com um programa muito básico, um de cada é suficiente.

Vamos ver como implementá-los.

# 4.1. Módulo
Para criar um módulo, precisamos anotar a classe com a anotação @Module. Essa anotação indica que a classe pode disponibilizar dependências para o contêiner:

```
@Module
public class VehiclesModule {
}
```

Então, precisamos adicionar a anotação @Provides nos métodos que constroem nossas dependências:

```
@Module
public class VehiclesModule {
    @Provides
    public Engine provideEngine() {
        return new Engine();
    }

    @Provides
    @Singleton
    public Brand provideBrand() { 
        return new Brand("isaccanedo"); 
    }
}
```

Além disso, observe que podemos configurar o escopo de uma determinada dependência. Nesse caso, damos o escopo do singleton à nossa instância Brand para que todas as instâncias de carro compartilhem o mesmo objeto de marca.

# 4.2. Componente
Continuando, vamos criar nossa interface de componente. Esta é a classe que irá gerar instâncias de Car, injetando dependências fornecidas por VehiclesModule.

Simplificando, precisamos de uma assinatura de método que retorne um Car e precisamos marcar a classe com a anotação @Component:

```
@Singleton
@Component(modules = VehiclesModule.class)
public interface VehiclesComponent {
    Car buildCar();
}
```

Observe como passamos nossa classe de módulo como um argumento para a anotação @Component. Se não fizéssemos isso, Dagger não saberia como construir as dependências do carro.

Além disso, como nosso módulo fornece um objeto singleton, devemos dar o mesmo escopo ao nosso componente porque Dagger não permite que componentes sem escopo se refiram a ligações com escopo.

# 4.3. Código do cliente

Finalmente, podemos executar o mvn compile para acionar os processadores de anotação e gerar o código do injetor.

Depois disso, encontraremos nossa implementação de componente com o mesmo nome da interface, apenas prefixado com “Dagger“:

```
@Test
public void givenGeneratedComponent_whenBuildingCar_thenDependenciesInjected() {
    VehiclesComponent component = DaggerVehiclesComponent.create();

    Car carOne = component.buildCar();
    Car carTwo = component.buildCar();

    Assert.assertNotNull(carOne);
    Assert.assertNotNull(carTwo);
    Assert.assertNotNull(carOne.getEngine());
    Assert.assertNotNull(carTwo.getEngine());
    Assert.assertNotNull(carOne.getBrand());
    Assert.assertNotNull(carTwo.getBrand());
    Assert.assertNotEquals(carOne.getEngine(), carTwo.getEngine());
    Assert.assertEquals(carOne.getBrand(), carTwo.getBrand());
}
```

# 5. Analogias da mola
Aqueles familiarizados com o Spring podem ter notado alguns paralelos entre os dois frameworks.

A anotação @Module de Dagger torna o contêiner ciente de uma classe de uma maneira muito semelhante a qualquer uma das anotações de estereótipo do Spring (por exemplo, @Service, @ Controller…). Da mesma forma, @Provides e @Component são quase equivalentes a @Bean e @Lookup do Spring, respectivamente.

Spring também tem sua anotação @Scope, correlacionada a @Singleton, embora observe aqui outra diferença em que Spring assume um escopo singleton por padrão, enquanto Dagger assume o que os desenvolvedores Spring podem se referir como escopo de protótipo, invocando o método do provedor cada vez dependência é necessária.

# 6. Conclusão
Neste artigo, explicamos como configurar e usar o Dagger 2 com um exemplo básico. Também consideramos as diferenças entre injeção em tempo de execução e em tempo de compilação.
