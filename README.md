# playwright-ts-bdd 

## 1. Instalação
1.1. Instalar o Playwright (com TypeScript): 
```
npm init playwright@latest
```

1.2. Instalar o Cucumber: 
```
npm install -D @cucumber/cucumber
```

1.3. Instalar o typescript: 
```
npm install -D typescript
```

1.4. Instalar o ts-node: 
```
npm install -D ts-node
```

### Explicações:
> O parâmetro -D instala como devDependency no package.json. Isso significa que num projeto completo, essas dependências não serão compiladas para gerar o “executável” do projeto. Como estamos rodando testes, não existe “executável” neste projeto, portanto essas dependências devem ficar nesta sessão.
precisamos instalar o typescript e o ts-node pois eles farão a tradução do TypeScript para JavaScript para que o Cucumber consiga interpretar os comandos.

<br>

## 2. Configuração
2.1. Criar o tsconfig.json na raiz do projeto: 
```
npx -p typescript tsc --init
```

2.2. Abrir o <b>tsconfig.json</b> e fazer as alterações:
- Remover o comentário de "allowSyntheticDefaultImports": true,
- Remover o comentário de "resolveJsonModule": true,
- Ajustar o target para “ESNext”: "target": "ESNext",

2.3. Criar o arquivo <b>cucumber.mjs</b> na raiz do projeto:
```
export default {
  format: ['html:bdd-tests/reports/cucumber-report.html'],
  parallel: 2,
  paths: ['bdd-tests/features/**/*.feature'],
  require: ['bdd-tests/step-definition/**/*.ts','bdd-tests/support/**/*.ts'],
  requireModule: ['ts-node/register'],
}
```
<br>

### Explicações:
> No arquivo tsconfig.json ajustamos as configurações para a compilação ser realizada de forma correta já que a sintaxe muda dependendo de qual versão do JavaScript você utiliza. Para mais informações veja aqui.
No arquivo cucumber.mjs, definimos o formato do report e o local onde será gravado, quantos testes em paralelo, o caminho dos feature files, o caminho dos step definition files e dos arquivos de suporte e fazemos o require do ts-node para ser utilizado durante a compilação. Para mais configurações [veja aqui](https://github.com/cucumber/cucumber-js/blob/main/docs/configuration.md).
A extensão .mjs significa “(M)odular (J)ava(S)cript”. E utiliza módulos ECMAScript ao invés de módulos CommonJS. A principal diferença é o uso de import & export (no ECMAScript) ao invés de require & module.exports (no CommonJS).

Fonte: https://codingforseo.com/blog/mjs-vs-cjs-files/

<br>

## 3. Implementação dos arquivos de teste

3.1. Criar o Feature file: 
> /bdd-tests/features/example.feature
```
Feature: Playwright website

Scenario: Has Title
    Given I am at the playwright website
    # When I open the page
    Then the title has the text "Playwright"

Scenario: Get Started Link
    Given I am at the playwright website
    When I click at "Get Started"
    Then the URL has the text "intro"
```

O primeiro cenário abre a página do Playwright e verifica se o título tem a palavra “Playwright”.
O segundo cenário abre a página do Playwright, clica no botão “Get Started” e confirma se a URL tem o texto “intro”.
Estes são os mesmos cenários implementados no arquivo /tests/example.spec.ts que é criado ao instalar o Playwright.


3.2. Criar o arquivo que integra Playwright e Cucumber: 
> /bdd-tests/support/custom-world.ts
```
import { setWorldConstructor, World, IWorldOptions } from '@cucumber/cucumber';
import { BrowserContext, Page, PlaywrightTestOptions } from '@playwright/test';

export interface CucumberWorldConstructorParams {
  parameters: { [key: string]: string };
}

export interface ICustomWorld extends World {
  context?: BrowserContext;
  page?: Page;
  playwrightOptions?: PlaywrightTestOptions;
}

export class CustomWorld extends World implements ICustomWorld {
  constructor(options: IWorldOptions) {
    super(options);
  }
}

setWorldConstructor(CustomWorld);
```

O Cucumber utiliza a classe World. Esta classe possui um escopo isolado e exposto para cada step e hooks através do “this“. Ou seja, permite o reuso de variáveis e métodos porém utiliza uma nova instância para cada execução de teste.

Neste arquivo, estamos criando uma interface ICustomWorld para ser usada nos testes e ter acesso ao BrowserContext, Page e PlaywrightTestOptions do Playwright, permitindo interagir com o browser e seus elementos.

3.3. Criar o arquivo que iniciará e encerrará o browser: 
> bdd-tests/support/hooks.ts
```
import { Before, After, BeforeAll, AfterAll } from '@cucumber/cucumber';
import { chromium, ChromiumBrowser } from '@playwright/test';
import { ICustomWorld } from './custom-world';

declare global {
    var browser: ChromiumBrowser;
}

BeforeAll(async function () {
    global.browser = await chromium.launch({
        headless: false,
    });
});

AfterAll(async function () {
    await global.browser.close();
});

Before(async function (this: ICustomWorld) {
    this.context = await global.browser.newContext();
    this.page = await this.context?.newPage();
});

After(async function (this: ICustomWorld) {
    await this.page?.close();
    await this.context?.close();
});
```

Neste caso iremos utilizar somente o chromium para simplificar o exemplo. Mas você pode adicionar quaisquer browser que desejar (que seja suportado pelos frameworks). Veja as Referências no final do artigo para exemplos.

3.4. Criar o step definition file: 
> bdd-tests/step-definition/example.step.ts
```
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { ICustomWorld } from '../support/custom-world';

Given('I am at the playwright website', async function (this: ICustomWorld) {
    await this.page!.goto('https://playwright.dev/');
});
  
When('I click at "Get Started"', async function (this: ICustomWorld) {
    await this.page!.getByRole('link', { name: 'Get started' }).click();
});
  
Then('the title has the text "Playwright"', async function (this: ICustomWorld) {
    await expect(this.page!).toHaveTitle(/Playwrights/);//this line is intentionally failing so we can see the reports
});

Then('the URL has the text "intro"', async function (this: ICustomWorld) {
    await expect(this.page!).toHaveURL(/.*intro/);
});
```

Este arquivo contem as ações para executar os testes e as interações com o browser. Bem semelhante ao arquivo tests/example.spec.ts. O passo “the title has the text “Playwright”” foi implementado intencionalmente para falhar para que possamos ver o report.

Podemos ver que utilizamos o “this: ICustomWorld” como parâmetro de cada step. Desta forma, conseguimos interagir com o browser.

O símbolo ! depois do this.page é chamado “Non-null assertion operator” e quer dizer (no TypeScript) para o verificador de tipos que esta variável não será null ou undefined em momento algum e que ele deve não alertar sobre erros de tipagem (pois em alguns momentos o verificador não consegue identificar). Sem ele, teremos um erro. Veja mais aqui.

<br>

## 4. Execução dos testes
Vamos rodar os testes usando Cucumber. Em seu terminal, rode:
```
npx cucumber-js
```

Como configuramos no arquivo bdd-tests/support/hooks.ts para o modo headless: false, o browser irá abrir durante a execução dos testes. O resultado deve ser 1 teste passando e 1 teste falhando. O arquivo bdd-tests/reports/cucumber-report.html será criado e ao abri-lo, isso é o que veremos:


Não vou entrar em detalhes em relação ao relatório pois é possível utilizar outros reporters mais completos, porém é um relatório que temos acesso ao básico pelo menos 🙂

A execução foi bem rápida, e podemos ver que o step “Then the title has the text “Playwright”” falhou, como esperado.


<br>

## 5. Fonte
Blog: Testing with Renata
```
https://testingwithrenata.com/blog/test-automation/playwright-bdd-cucumber-e-a-minha-opiniao-sobre-isso/
```






