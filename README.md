# How to Suppress the `'varname' is declared but its value is never read.` Hint from `typescript-language-server`

## tldr
Set `compilerOptions.noUnusedLocals` to be `true` in you `tsconfig.json`.
```json
{
  "compilerOptions": {
    "noUnusedLocals": true
  }
}
```
Then add `// @ts-expect-error` above the line where you get the hint. 

## The Problem
If you have this piece of TypeScript code alongside an empty `tsconfig.json`

```typescript
function foo() {
  const bar = "bar";
}
```

and run an editor (I use [neovim](https://github.com/neovim/neovim) with
[vim-lsp](https://github.com/prabirshrestha/vim-lsp)) with
[typescript-language-server](https://github.com/typescript-language-server/typescript-language-server),
you're gonna get a diagnostic hint: `typescript:Hint:6133:'bar' is declared but
its value is never read.`

This is usually useful. I generally don't want to have unused local variables
in my code. However, there are cases where I do want them. In such cases, how
do I suppress the hint?

## Is eslint Giving the Hint?

This was the first thing I thought of. `tsc` doesn't complain about the code,
so I thought this isn't something that TypeScript itself doesn't like and
therefore has nothing to do with `tsconfig.json`. Also this doesn't look like
something from type-checking but seems more like linting. Running `eslint` on
this piece of code with recommended settings from `typescript-eslint` does say
that this is an error that violates `@typescript-eslint/no-unused-vars`, with a
similar message. It's also not unprecedented for language servers to take
advantage of popular linters when they are available.
[python-lanugage-server](https://github.com/palantir/python-language-server)
does that. So I did try to use `// eslint-disable` to suppress this. This
didn't work.

It took me some time to realize that, this hint has nothing to do with eslint.
Unlike `python-language-server`, `typescript-language-server` [does not use
eslint at
all](https://github.com/typescript-language-server/typescript-language-server/issues/708). 

## Is It TypeScript Itself?

Obviously, `typescript-language-server` reports things that violates the TypeScript
language itself. If under some settings, TypeScript doesn't allow unused variables,
it's not surprising that the language server hints about it.

The official compiler for TypeScript, `tsc`, does disallow unused variables
when the option `noUnusedLocals` is on. This is documented
[here](https://www.typescriptlang.org/tsconfig/#noUnusedLocals). The
documentation gives exactly the same text as I saw from the hint. If it also
has the number `6133`, It'd be certain that this is what I was looking for. It
doesn't though. `eslint`'s `no-unused-vars` rules also give messages that look
very similar, so before I managed to be sure that the language server has
nothing to do with `eslint`, I still always suspected it. `tsc --help --all`
will also give documentation about this option (`tsc --help` doesn't have it,
another thing that made it harder for me...):

```
--noUnusedLocals  Enable error reporting when local variables aren't read.
             type:  boolean
          default:  false
```

It's false by default. Makes sense, since `tsc` doesn't complain about our code
when run without this option used. Then why do I still get it with an empty
`tsconfig.json`?

If we try to explicitly disable this via `tsconfig.json`, we still get this hint.
After all, disabling something that is off by default doesn't change anything.

In addition, `// @ts-ignore` doesn't turn off the hint either.

Using `// @ts-expect-error` on this line is an error. This actually makes
sense, without `noUnusedLocals` set, `tsc` doesn't consider having unused
variables an error, so it's indeed wrong to use `@ts-expect-error` here. This
is probably also why `@ts-ignore` has no effect here. There isn't an error,
what do you want it to ignore?

How annoying, if the language doesn't think this is an error, why does the
language server insist on reporting it!

All these things with `tsc` made it seem like the hint has nothing to do
with the language itself, making it so hard for me to find the solution.

## The Solution

So we know `// @ts-expect-error` errors out because it doesn't consider having
unused local variables an error. Then what if we force it to do so? We know that
`noUnusedLocals` is false by default, and the language server hints it despite
that. Making `noUnusedLocals` false won't change anything, then what if we make
it true?

```json
{
  "compilerOptions": {
    "unUsedLocals": true
  }
}
```

After see `@ts-ignore` not having any effect, I doubted that this changes anything.
But it does!

`typescript:Hint:6133:'bar' is declared but its value is never read.`

is now

`typescript:Error:6133:'bar' is declared but its value is never read.`

Note that the word "Hint" from the former is replaced by "Error" in the latter.

So the language server is respecting `tsconfig.json`. When `tsconfig.json` says
this isn't an error, it doesn't consider it an error. But it considers it a
hint!

TypeScript provides things like `@ts-ignore` for errors, but not for hints, as
for the language itself, there isn't a concept called "hint". It's purely a
language server thing. And the langauge server didn't make it easy for us to
understand what a hint is, to which options it relates and how it can be turned
off. Perhaps it's nicer to VSCode users? I don't know.

Fortunately, now we know the distinction between a hint and an error for the language
server, and how to make a hint an error. Then we can just use how we suppress errors
on it: `// @ts-expect-error`.

So to turn off the language server complaining about unused variables, we need to
firstly turn it on!

I love how it is harder to suppress a hint than an error here.
