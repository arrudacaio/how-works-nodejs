# Como funciona o event loop no Node.js

## Event loop
O event loop é um loop infinito com condição de saída, ou seja, podemos entender o event loop como uma estrutura similar a um `while (true)` que monitora constantemente a **Call Stack** e outras filas de tarefas. Ele é o mecanismo central que permite ao Node.js realizar operações não-bloqueantes apesar de ser single-threaded.

## Call Stack
É uma estrutura de dados do tipo stack (pilha) que armazena os contextos de execução (funções que foram invocadas para serem executadas) do seu programa JavaScript.

Por exemplo, considere o código abaixo:
```js
const multiply = (a, b) => a * b;
const square = (number) => multiply(number, number);
const printSquare = (number) => console.log(square(number));

printSquare(5);
```

No momento em que executamos esse código, a **Call Stack** vai ser preenchida da seguinte forma:

| Call Stack |
|------------|
| multiply   |
| square     |
| printSquare|
| main       |
|            |

O event loop vai então processar e remover da stack as funções na ordem inversa da qual foram adicionadas:

1. `multiply` é executada e retorna o resultado
2. `square` recebe o resultado e é removida
3. `printSquare` recebe o resultado e é removida
4. `main` (contexto global) é removido

Após isso, se não houver mais nada para executar, o processo do Node.js será encerrado. Este processo que acabei de descrever é síncrono e bloqueante, pois no código demonstrado não existe nenhuma operação assíncrona.

Portanto, sempre que existir algum item (contexto de execução) na Call Stack, nenhum outro processo está sendo executado no Node.js, apenas este item. Isso ocorre porque o Node.js opera em uma única thread principal. É por causa deste comportamento que dizemos que operações síncronas são bloqueantes.

## Task Queues

Diferente do exemplo anterior com apenas operações síncronas, quando nosso programa contém chamadas assíncronas, o event loop passa a utilizar diferentes filas de tarefas (Task Queues) para gerenciar a execução.

No Node.js, existem várias filas de tarefas, com diferentes prioridades:

1. **Timer Queue**: Armazena callbacks de `setTimeout()` e `setInterval()`
2. **I/O Queue**: Contém callbacks de operações de I/O (leitura/escrita de arquivos, requisições de rede, etc.)
3. **Check Queue**: Guarda callbacks agendados pelo `setImmediate()`
4. **Close Queue**: Contém callbacks de eventos de "close" (por exemplo, quando um socket é fechado)

Vamos ver como funciona com um exemplo:

```js
console.log("Início");

setTimeout(() => {
  console.log("Timeout executado");
}, 0);

fs.readFile('arquivo.txt', () => {
  console.log("Arquivo lido");
});

setImmediate(() => {
  console.log("SetImmediate executado");
});

console.log("Fim");
```

O fluxo de execução seria:

1. "Início" é impresso (operação síncrona)
2. `setTimeout` é registrado e seu callback vai para a Timer Queue
3. `fs.readFile` é iniciado e seu callback será adicionado à I/O Queue quando a leitura for concluída
4. `setImmediate` é registrado e seu callback vai para a Check Queue
5. "Fim" é impresso (operação síncrona)
6. Como a Call Stack está vazia, o event loop verifica as filas, seguindo sua ordem de prioridade
7. O callback de `setTimeout` é executado, imprimindo "Timeout executado"
8. O callback de `setImmediate` é executado, imprimindo "SetImmediate executado"
9. Quando a leitura do arquivo terminar, o callback de `fs.readFile` é executado, imprimindo "Arquivo lido"

O ponto-chave aqui é entender que o event loop só processa itens das Task Queues quando a Call Stack está vazia. Isso significa que operações assíncronas não bloqueiam a execução do programa, permitindo que outras tarefas sejam realizadas enquanto aguardamos resultados de operações potencialmente demoradas.

### Fases do Event Loop

O event loop segue um ciclo definido de fases, cada uma com sua própria fila:

1. **Timers**: Processa callbacks de `setTimeout()` e `setInterval()`
2. **Pending callbacks**: Executa callbacks de operações do sistema adiadas
3. **Idle, prepare**: Usado internamente pelo Node.js
4. **Poll**: Busca novos eventos de I/O e processa seus callbacks
5. **Check**: Executa callbacks registrados via `setImmediate()`
6. **Close callbacks**: Executa callbacks de eventos "close"

O event loop prioriza a execução de tarefas na Call Stack. Somente quando a Call Stack está vazia, ele passa a verificar as Task Queues, respeitando a ordem de fases descrita acima.

### Microtasks

Além das Task Queues mencionadas, o Node.js também tem filas de microtasks:

1. **Next Tick Queue**: Para callbacks registrados via `process.nextTick()`
2. **Promise Queue**: Para handlers de promises (`.then()`, `.catch()`, etc.)

As microtasks têm prioridade especial: são processadas imediatamente após a conclusão da operação atual na Call Stack e antes de continuar para a próxima fase do event loop. Isso significa que `process.nextTick()` e promises têm prioridade sobre outras operações assíncronas.

```js
console.log("Início");

setTimeout(() => console.log("Timeout"), 0);

Promise.resolve().then(() => console.log("Promise"));

process.nextTick(() => console.log("NextTick"));

console.log("Fim");
```

Saída:
```
Início
Fim
NextTick
Promise
Timeout
```

Esta priorização de microtasks é fundamental para entender o comportamento do Node.js em aplicações complexas com múltiplas operações assíncronas.
