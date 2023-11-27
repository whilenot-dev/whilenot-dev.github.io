+++
title = "Template metaprogramming with TypeScript"
date = "2023-11-14T14:33:56+01:00"
tags = ["typescript", "metaprogramming", "template metaprogramming"]
description = "Generate valid source code programmatically"
+++

# Generate valid source code programmatically

It's about time for another blog post... been a while! While I still owe it to myself to write the 2nd part of the [previous blog post](https://blog.whilenot.dev/posts/cool-bugs-vs-shitty-bugs-part-1/) (does anybody have tips for shitty bugs?), I thought it would be nice for a completely different topic - Metaprogramming.

Sometimes writing source code gets a bit boring and repetitive. Anytime I encounter such a feeling I reflect on my life and how I never wanted to do any assembly-line work ever again. We all work with software for different use cases, but one thing we all probably have in common is the desire to automate the boring and repetitive tasks.

In this blog post I want to show you how one can create _TypeScript_ files programmatically via _JavaScript_ scripts. Yes, you read that correctly - I'm using _JavaScript_ scripts to generate valid _TypeScript_ files, which in turn should each transpile to valid _JavaScript_ again. I'll provide some methods I got to know (and use) for _TypeScript_ source code generation and want to elaborate a bit on their tradeoffs.

## When to generate (source) code

The communication from your "core" program to an "outer world" can include a lot of repetition. In cases where the abstraction into functions doesn't really cut it, code generation can be highly beneficial for those modules of a program that communicate with that "outer world". Any changes in the "outer world" should be - in the best case - covered by automation and just one command away from getting in sync with your program again. I understand source code generation here as the automated output of a processed [contract](<(https://en.wikipedia.org/wiki/Design_by_contract)>).

An "outer world" and its "contract" to describe its interface can include schemas in a database, any GraphQL or REST API (OpenAPI), or any kind of data structure that is following a certain schema definition (eg. [Directus CMS](https://docs.directus.io/reference/system/schema.html)). In the best case, the schema's specification is well-defined and publicly available, so it'll even be a nice exercise to provide your own generator for another kind of output. In this post I want to keep it simple and show the generation of valid _TypeScript_ source code from an ill defined JSON file.

Before working on a new generator script I would always strive to look for existing and maintained projects. There are many code generators available for various languages and use cases:

- [quicktype](https://app.quicktype.io/)
- [swagger-codegen](https://github.com/swagger-api/swagger-codegen)
- [openapi-generator-cli](https://openapi-generator.tech/)
- [prisma](https://www.prisma.io/docs/concepts/components/prisma-schema/generators)
- etc.

Make sure to check for existing code generator tools or language parsers whenever the work on a project becomes a bit boring.

![meme_automation.jpg](/images/template-metaprogramming-with-typescript/meme_automation.jpg "Automate all the things")

That being said, let me show you some methods to generate _TypeScript_ source code...

## Source code generation methods

You can find everything mentioned here in the accompanied [GitHub repository](https://github.com/whilenot-dev/template-metaprogramming-with-typescript).

For the sake of this post I'm making use of an ill defined JSON as data input:

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

(you can probably already imagine better data with a well-defined schema as input)

...and want to generate the following _TypeScript_ source code file by executing a _JavaScript_ script:

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

My methods include source code generation via:

1. [Template literals](#1-template-literals)
1. [A template engine](#2-a-template-engine)
1. [A writer library](#3-a-writer-library)
1. [The _TypeScript_ compiler API](#4-the-_typescript_-compiler-api)

Ready? So let's start with the easiest one...

### 1. Template literals

![meme_level_1.jpg](/images/template-metaprogramming-with-typescript/meme_level_1.jpg "Level 1")

This one's a _JavaScript_ script for source code generation via template literals:

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

As you can probably already see, this method is very error prone and not very pleasant to maintain. I'm basically concatenating strings one after the other and need to rely on good naming conventions to make sense of the generated source code parts. Quotes, identation and formatting become manual work and affect readability.

I can already predict the future struggles to maintain something like this. It's probably for the best to use this method only for the most simple repetitions (this example here being one of them).

### 2. A template engine

![meme_level_2.jpg](/images/template-metaprogramming-with-typescript/meme_level_2.jpg "Level 2")

This one's a _JavaScript_ script for source code generation via a template engine (eg. [handlebars](https://handlebarsjs.com/)):

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

This method is pretty self-exlanatory: I pick my template engine of choice, add it to my dependencies, compile a template, and fill in the values for the template variables.

Depending on the complexity of the generated output, a template engine can provide enough built-in functionalty to iterate over code parts and in general provides good readability for any future maintainance. There are template engines available for all major programming languages: `Python` has [`Jinja`](https://jinja.palletsprojects.com/en/3.1.x/), `Node.js` has [`handlebars`](https://handlebarsjs.com/), `Go` has [`pongo2`](https://www.schlachter.tech/solutions/pongo2-template-engine/), among others.

Did you notice how the template engine needed to define its own syntax for basic programming constructs (eg. for-loops)? Sometimes those built-in's are not enough for the complexity of a generated source code. While the template engine of choice might provide functionality to register custom helpers (eg. like with [handlebars](https://handlebarsjs.com/guide/expressions.html#helpers)), you might feel that the fight for readability seems like a lost cause in the long run.

I personally prefer to avoid any unnecessary dependencies, but if a template engine takes already part in your dependency list, I'd say it's a good extension to the first method above. In more complex cases you might want to make better use of the capabilities of your scripting language...

### 3. A writer library

![meme_level_3.jpg](/images/template-metaprogramming-with-typescript/meme_level_3.jpg "Level 3")

```javascript
import CodeBlockWriter from "code-block-writer";

import data from "../data/input.json" assert { type: "json" };

main();

function main() {
  const writer = new CodeBlockWriter({
    indentNumberOfSpaces: 2,
    newLine: "\n",
    useSingleQuote: true,
    useTabs: false,
  });

  data.forEach((val) => {
    writer.write(`interface SomeInterface${val.name}`).block(() => {
      writer.writeLine(`discriminator: "${val.name}";`);
      writer.writeLine(`type: ${val.type};`);
    });
    writer.blankLine();
  });
  writer.writeLine(
    `export type SomeInterface = ${data
      .map((val) => `SomeInterface${val.name}`)
      .join(" | ")};`
  );

  const result = writer.toString();

  console.log(result);
}
```

You might feel that the programming constructs of a template engine are not enough for your use case. In that case you want to make use of your scripting language to its full extend and only want to abstract certain output blocks.

TODO: ...

In any other cases it might be better to define your output on AST-level and let a suited compiler "unparse" it...

### 4. The _TypeScript_ compiler API

![meme_level_4.jpg](/images/template-metaprogramming-with-typescript/meme_level_4.jpg "Level 4")

Oki, so you know that you can transpile `.ts`-files into `.js`-files with the _TypeScript_ compiler (`tsc`). Did you also know that the `typescript` package comes with factory functions that provide you with everything needed to create an _TypeScript_ [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) with plain _JavaScript_ programmatically?

This is how a _JavaScript_ script for source code generation via the [_TypeScript_ compiler API](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API) would look like:

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

Granted, this one's overkill for our use case here. But there are times when you want some additional programming functionality - like variables - to refer to other parts of the generated source code easily. As far as I know this is the only method that handles literal strings (through AST nodes, eg. `createStringLiteral`) and formatting (through a "printer", sometimes referred to as "unparser").

While the documentation for the [compiler API](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API) seems kinda sparse, it's quite straight forward to just type out parts of the wanted output and then head over to the [_AST Explorer_](https://astexplorer.net/) to inspect the resulting [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree). Just make sure to select the `typescript` parser from the dropdown in the navigation bar:

![astexplorer.gif](/images/template-metaprogramming-with-typescript/astexplorer.gif "AST Explorer")

By scanning through the AST tree one will see that names in the _AST Explorer_'s tree-nodes match the method names of the compiler API closely.

## A word on generated source code and `git`

During my work for clients and companies I could see some inconsistencies when it comes to the integration of generated source code with the rest of the code base. One practice is to run the code generation manually by a developer and add the generated output to the [VCS](https://en.wikipedia.org/wiki/Version_control) (eg. `git`) via a PR; another is to ignore it by the VCS, treat code generation as additional step in the automated build pipeline and handle it like any other build artifact.

My preference of whether or not to add generated source code artifacts to a VCS depends mainly on the following questions:

- Is the generated output human readable?
- Is the fetching of input data considered secure and trustable?
- Is the spec for input data considered stable?
- Is the generated output tested?

No matter how hard you try, a code-generating script can become buggy and requires maintenance. In such cases it is always beneficial to jump just into the generated output, fix existing bugs right there and then, and fix the code-generating script whenever there's some free time later on. Human readable code is easily hackable, and that's a big advantage!

Human readable source code won't take up much space in your repository anyway and it might be beneficial to ask for peer reviews on generated, human readable output.

In other cases VCSs aren't well suited for binary files or even minified output. If readability is already a lost cause, code generation can become a build artifact like any other. The challenge then is to provide a stable pipeline with a certain amount of guarantee for the artifacts integrity.

## Conclusion

In this post we barely scratched the surface of template metaprogramming.

Metaprogramming lets you work closely with the basic building blocks of a programming language where source code becomes data again. If you never wrote a source code generator before, I would highly recommend it to you. Metaprogramming is one of the neatest programming techniques and can safe you a bunch of time.

With all the hype around LLMs it might be worthwhile add code generation via generative AI to the methods listed here, but that's something for later maybe.

Thank you for having a look and stepping through it with me!
