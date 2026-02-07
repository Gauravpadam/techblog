---
title: "LocalCode: Building the Code Harness"
date: 2026-02-07
draft: false
tags: ["java", "design-patterns", "architecture", "code-generation"]
description: "The ideation, the thought process, the difficulties and the systems thinking involved in making the parser for LocalCode problems."
---

I had a realisation. This won't work.

Every time I added a problem to LocalCode, I had to write parsing code. For inputs. For outputs. For every single problem.

And that sucks.

## Why this was a problem

The parsing logic is almost always the same. An integer array is parsed the same way whether it's for Two Sum or Maximum Subarray. I was writing the same code over and over, just with different variable names.

If you're building a coding platform and you find yourself copy-pasting parsing code for the 13th time, something has gone wrong.

So I decided to generate it internally. The backend would figure out what types a problem needs and produce the parsing code automatically. Something like a *harness* that wraps around the user's code.

## First attempt: Store the types in the database

My first thought: store the input and output types at the problem level. Add a few columns to the `Problem` table, read them when generating code.

Not an issue, I said. So I went ahead with this design.

To mention early: I smelled something fishy. I was already storing starter code, and now I'm asking users for "please also give me the input types and output types." That seemed... redundant. But I had to start implementing, so I pushed the thought aside.

I wrote generators for common datatypes. Read types from the database, mapped them to enums. Life was great.

Then I remembered there could be multiple parameters.

So now I need the input type for *each* parameter. Another thing to ask from whoever's preparing the problem. And I still need the output type. Yet another field.

What smells fishy, reeks later.

## The better solution

I stepped back and looked at what I already had.

The starter code.

```java
public int[] twoSum(int[] nums, int target)
```

The return type is right there. The parameter types are right there. The method name is right there. Everything I needed was already in the signature.

So I wrote a parser instead:

```java
private MethodSignature parseStarterCode(String starterCode) {

    Pattern pattern = Pattern.compile(
        "(?:public|protected|private|static|final|\\s)*" +
        "([\\w<>\\[\\]]+)\\s+" +        // return type
        "([a-zA-Z_]\\w*)\\s*" +         // method name
        "\\(([^)]*)\\)"                 // params
    );

    Matcher matcher = pattern.matcher(starterCode);

    if (!matcher.find()) {
        throw new IllegalArgumentException("No method signature found");
    }

    String returnType = matcher.group(1);
    String methodName = matcher.group(2);
    String paramsStr = matcher.group(3).trim();

    // ... parse params ...

    return new MethodSignature(returnType, methodName, params);
}
```

The regex captures three groups: return type, method name, and the parameters string. Then I split the parameters and extract each one's type and name.

No extra database columns. No asking users for information they've already provided.

Starter code became my single source of truth. BOOM.

## But how do I structure the generated code?

I was still running in circles on the structure. How exactly do I stitch the generated code with the user's code?

So I did what any reasonable person would do. I opened HackerRank and looked at their network tab.

And there it was. Three pieces of code in the request payload:

- **Head** — imports, helper methods
- **Problem code** — the user's solution, untouched
- **Tail** — the main method that reads input, calls the function, prints output

That's it. That was the structure I needed.

I provide the Head and the Tail. The user provides the Problem code. Sandwich them together, send to the container.

Sometimes the best documentation is someone else's network requests.

## Making it work for multiple languages

I had it working for Java. But LocalCode has to support many programming languages.

This calls for an interface.

```java
public interface CodeEmitter {
    String generateImports();
    String generateTailCode(String methodToCall);
    String generateInputParsing(DataType dataType);
    DataType dataTypeMap(String paramType);
}
```

Each language gets its own implementation: `JavaCodeEmitter`, `PythonCodeEmitter`, `JSCodeEmitter`.

A factory picks the right emitter based on the submission language:

```java
@Service
public class EmitterFactory {
    private final PythonCodeEmitter pythonCodeEmitter;
    private final JSCodeEmitter jsCodeEmitter;
    private final JavaCodeEmitter javaCodeEmitter;

    // constructor with DI ...

    public CodeEmitter getEmitter(String language) {
        return switch (language.toLowerCase()) {
            case "java" -> javaCodeEmitter;
            case "python" -> pythonCodeEmitter;
            case "javascript", "js" -> jsCodeEmitter;
            default -> throw new IllegalArgumentException("Unsupported language: " + language);
        };
    }
}
```

Factory makes the emitter. Emitter makes the code. Each class has one job. The Gang of Four would approve.

## The pipeline

The `CodeHarness` service ties it all together:

```java
@Service
public class CodeHarness {

    private final EmitterFactory emitterFactory;

    public CodeHarness(EmitterFactory emitterFactory) {
        this.emitterFactory = emitterFactory;
    }

    public String generate(ExecutionRequest request) {
        CodeEmitter emitter = emitterFactory.getEmitter(request.getLanguage());
        StringBuilder harness = new StringBuilder();

        harness.append(emitter.generateImports());
        harness.append(emitter.generateTailCode(request.getMethodToCall()));

        return harness.toString();
    }
}
```

Now the pipe was clean:

1. For head code: generate imports
2. For tail code: parse the method signature, figure out the types, generate the parsing logic
3. Combine head + user code + tail
4. Send to container

Great stuff.

## Testing (a.k.a. things breaking)

Then I ran it against all 13 seeded problems.

Things broke. Empty lines caused exceptions. Some arrays needed special output formatting. Nested generics like `List<List<Integer>>` broke the parameter splitting because commas exist inside angle brackets too.

That last one was fun. The fix:

```java
if (!paramsStr.isEmpty()) {
    int depth = 0;
    int start = 0;

    for (int i = 0; i < paramsStr.length(); i++) {
        char c = paramsStr.charAt(i);

        if (c == '<') depth++;
        else if (c == '>') depth--;
        else if (c == ',' && depth == 0) {
            extractParam(paramsStr.substring(start, i), params);
            start = i + 1;
        }
    }

    // last param
    extractParam(paramsStr.substring(start), params);
}
```

Track the angle bracket depth. Only split on commas when depth is zero. Simple once you see it, annoying until you do.

I fixed the issues one by one. Quick patches for now. Custom types like `TreeNode` will need a bigger refactor — that's for the next iteration.

## That's it

The harness now generates all the parsing code automatically. I add a problem, write the starter code, and the backend figures out the rest.

The code is in the LocalCode repo if you want to poke around.

What smells fishy, reeks later. But if you catch it early enough, you can pivot before it stinks up the whole codebase.
