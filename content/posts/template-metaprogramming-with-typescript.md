+++
title = "Template metaprogramming with TypeScript"
date = "2023-11-14T14:33:56+01:00"
tags = ["typescript", "metaprogramming", "template metaprogramming"]
description = "Generate valid TypeScript source code programmatically"
+++

# Generate valid TypeScript source code programmatically

It's about time for another blog post... been a while! While I still owe it to myself to write the 2nd part of the [previous blog post](https://blog.whilenot.dev/posts/cool-bugs-vs-shitty-bugs-part-1/) (does anybody have tipps for shitty bugs?), I thought it would be nice for a completely different topic - Metaprogramming.

Sometimes programming gets a bit boring and repetitive. Anytime I encounter such a feeling I think about my life and how I never wanted to do any repetitive labor. We all work with software for different use cases, but one thing we all probably have in common is the desire to automate the boring repetitive tasks.

In this blog post I want to show you how one can create TypeScript files programmatically via JavaScript scripts. Yes, you read that correctly - I'm using JavaScript scripts to generate valid TypeScript files, which in turn should each transpile to valid JavaScript again. I'll provide different methods for TypeScript source code generation and want to elaborate a bit on each method with its tradeoff.

For the sake of this post I'm making use of a generic JSON as input (you can probably already immagine more worthwhile input data):

```json
// data/input.json
[
  {
    "name": "A",
    "type": "boolean"
  },
  {
    "name": "B",
    "type": "number"
  },
  {
    "name": "C",
    "type": "string"
  }
]
```

...and want to generate the following TypeScript source code file by executing a JavaScript script:

```typescript
// output.ts
interface SomeInterfaceA {
  discriminator: "A";
  type: boolean;
}

interface SomeInterfaceB {
  discriminator: "B";
  type: number;
}

interface SomeInterfaceC {
  discriminator: "C";
  type: string;
}

export type SomeInterface = SomeInterfaceA | SomeInterfaceB | SomeInterfaceC;
```

You can find everything mentioned here in the accompanied [GitHub repository](https://github.com/whilenot-dev/template-metaprogramming-with-typescript).

## Source code generation methods

My methods include source code generation via:

1. [Template literals](#1-template-literals)
1. [A template engine](#2-a-template-engine)
1. [The TypeScript compiler API](#3-the-typescript-compiler-api)
1. [LLMs](#4-llms) (maybe)

Ready? So let's start with the easiest one...

### 1. Template literals

This one's a JavaScript script for source code generation via template literals:

```javascript
import data from "../data/input.json" assert { type: "json" };

main();

function main() {
  const interfaces = data
    .map(
      (val) => `interface SomeInterface${val.name} {
  discriminator: "${val.name}";
  type: ${val.type};
}
`
    )
    .join("\n");
  const union =
    "export type SomeInterface = " +
    data.map((val) => `SomeInterface${val.name}`).join(" | ") +
    ";\n";

  const result = [interfaces, union].join("\n");

  console.log(result);
}
```

As you can probably already immagine, this method is very error prone and not very pleasant to maintain. I'm basically concatenating some strings and need to rely on good naming conventions to make sense of the generated source code parts. I can already predict the future struggles to maintain something like this.

It's probably for the best to use this method of source code generation only for the most simple repetitions, this example here being one of them.

### 2. A template engine

This one's a JavaScript script for source code generation via a template engine (eg. [handlebars](https://handlebarsjs.com/)):

```javascript
import Handlebars from "handlebars";

import data from "../data/input.json" assert { type: "json" };

main();

function main() {
  const source = `{{#data}}
interface SomeInterface{{name}} {
  discriminator: "{{name}}";
  type: {{type}};
}

{{/data}}
export type SomeInterface = {{#data}}{{#if @first}}{{else}} | {{/if}}SomeInterface{{name}}{{/data}};
`;

  const template = Handlebars.compile(source);
  const result = template({ data });

  console.log(result);
}
```

This one's pretty self-exlanatory: I pick my template engine of choice, add it to my dependencies, compile a template, and fill in the values for the template variables. Depending on the complexity of the generated output, a template engine can provide enough built-in functionalty to iterate over code parts and, in general, provides a good readability for any future maintainance.

Did you notice how the template engine needed to define its own syntax for basic programming constructs (eg. for-loops)? Sometimes those built-in's are not enough for the complexity of a source code generation. While the template engine of choice might provide functionality to register custom helpers (eg. like [handlebars](https://handlebarsjs.com/guide/expressions.html#helpers)), you might feel like the fight for readability is a lost cause in the long run.

I personally prefer to avoid any unnecessary dependencies, but if a template engine takes already part in your dependency list, I'd say go for it!

In any other cases it might be better to define your output on AST-level and let a suited compiler "unparse" it...

### 3. The TypeScript compiler API

Oki, so you know that you can transpile `.ts`-files into `.js`-files with the TypeScript compiler (`tsc`). Did you also know that the `typescript` package comes with factory functions that provide you with everything needed to create a TypeScript [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) with plain JavaScript programmatically?

This is how a JavaScript script for source code generation via the [TypeScript compiler API](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API) would look like:

```javascript
import ts from "typescript";

import data from "../data/input.json" assert { type: "json" };

main();

function main() {
  const statements = [
    ...data.map(({ name, type }) =>
      ts.factory.createInterfaceDeclaration(
        undefined,
        `SomeInterface${name}`,
        undefined,
        undefined,
        [
          ts.factory.createPropertySignature(
            undefined,
            "discriminator",
            undefined,
            ts.factory.createStringLiteral(name)
          ),
          ts.factory.createPropertySignature(
            undefined,
            "type",
            undefined,
            ts.factory.createIdentifier(type)
          ),
        ]
      )
    ),
    ts.factory.createTypeAliasDeclaration(
      [ts.factory.createToken(ts.SyntaxKind.ExportKeyword)],
      "SomeInterface",
      undefined,
      ts.factory.createUnionTypeNode(
        data.map((val) =>
          ts.factory.createIdentifier(`SomeInterface${val.name}`)
        )
      )
    ),
  ]
    .flatMap((val) => [ts.factory.createIdentifier("\n"), val])
    .slice(1);

  const sourceFile = ts.factory.createSourceFile(
    statements,
    ts.factory.createToken(ts.SyntaxKind.EndOfFileToken),
    ts.NodeFlags.None
  );
  const printerOptions = {
    newLine: ts.NewLineKind.LineFeed,
  };
  const printer = ts.createPrinter(printerOptions);
  const result = printer.printFile(sourceFile);

  console.log(result);
}
```

Granted, this one's overkill for our use case here. But there are times when you want some additional programming functionality - like variables - to refer to other parts of the generated source code easily.

While the documentation for the [compiler API](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API) seems kinda sparse, it's quite straight forward to just type out parts of your wanted output and then head over to the [AST Explorer](https://astexplorer.net/) and inspect the resulting [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree). Just make sure to select the `typescript` parser from the dropdown in the navigation bar:

![astexplorer.gif](/images/template-metaprogramming-with-typescript/astexplorer.gif "AST Explorer")

You'll see that the names in AST Explorer are also the names of the compiler API.

### 4. LLMs

With the hype of LLMs it might be worthwile to look into that... maybe, but maybe later.

## My personal practices

There are many code generators available:

[quicktype](https://app.quicktype.io/)
[dts-gen](https://github.com/microsoft/dts-gen)
[swagger-codegen](https://github.com/swagger-api/swagger-codegen)
[openapi-generator-cli](https://openapi-generator.tech/)

I would always strive to use an existing project before I make my own attempt.

### Why should I generate source code?

Source code generation is highly beneficial for those parts of your program that communicate with the "outer world". Any changes in the outer world should in the best case just be one executed command away from getting in synch with your own code. You should strive to stay in sync here and automate any synchronization to make it as fast as possible.

[Design by contract](https://en.wikipedia.org/wiki/Design_by_contract)

### Should I add my generated source code files to VCS?

Do I generate binary code? If so, then don't add it to your VCS, but instead include it in the build step of your pipeline. Otherwise just add the generated source code files to your VCS.

My reasoning here is the following:

- Binary files aren't very hackable. No matter how hard you try, you little code generating script can become buggy and requires maintenance. In such cases it is always worth it to just right into the generated output, fix existing bugs right there and now and then fix the generating script when there's time later on.
- Binary files aren't human readable, but source code should be human readable. If I tinker with template metaprogramming techniques and don't output human readable source code, then I didn't do my job really well.
- VCS aren't good with binary files and human readable source code won't take up much space in your repository.

## Conclusion

In this post we barely scratched the surface of template metaprogramming.

Metraprogramming...

If you never generated source code before, I would highly recommend it to you. Metaprogramming is one of the neatest programming techniques and can safe you a bunch of time.

Thank you for having a look and stepping through it with me!
