Hi @haiko, sorry this went unanswered for so long.

The answer is still yes for the current version: the compiler API works the same way, and the README section you linked has since been replaced by the docs site, which now has a full worked example:

- [Custom Translation Compiler recipe](https://ngx-translate.org/recipes/custom-compiler-guide/)
- [TranslateCompiler API reference](https://ngx-translate.org/reference/translate-compiler-api/)

The idea in short: a compiler rewrites your translation values once, when a language is loaded, and the app renders the rewritten values. Your JSON files stay untouched.

You extend `TranslateCompiler` and implement two methods:

```ts
import { Injectable } from '@angular/core';
import {
  TranslateCompiler,
  Language,
  TranslationObject,
  InterpolatableTranslationObject,
  InterpolateFunction,
} from '@ngx-translate/core';

@Injectable()
export class MyCompiler extends TranslateCompiler {
  // one translation value
  compile(value: string, lang: Language): string | InterpolateFunction {
    return value.toUpperCase();
  }

  // the whole file, nested objects included
  compileTranslations(translations: TranslationObject, lang: Language): InterpolatableTranslationObject {
    const compiled: InterpolatableTranslationObject = {};
    for (const [key, value] of Object.entries(translations)) {
      if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
        compiled[key] = this.compileTranslations(value as TranslationObject, lang);
      } else if (typeof value === 'string') {
        compiled[key] = this.compile(value, lang);
      } else {
        compiled[key] = value;
      }
    }
    return compiled;
  }
}
```

Then register it:

```ts
// standalone API, current versions
provideTranslateService({
  loader: provideTranslateHttpLoader(),
  compiler: provideTranslateCompiler(MyCompiler),
});

// NgModule API, the versions from 2022
TranslateModule.forRoot({
  loader: { provide: TranslateLoader, useFactory: HttpLoaderFactory, deps: [HttpClient] },
  compiler: { provide: TranslateCompiler, useClass: MyCompiler },
});
```

Two details that may be the confusing part:

- `compile()` can return a string or a function. Return a string for transformations that don't depend on interpolation params. Return a function when your syntax has placeholders that should be resolved at render time; that's how [ngx-translate-messageformat-compiler](https://github.com/lephyrus/ngx-translate-messageformat-compiler) compiles ICU MessageFormat patterns once per language load. The guide has a small example of both.
- If what you want is to preprocess the files themselves before the app ships, that's a build step, not a compiler. Since v18 you can also feed precompiled translations to `setCompiledTranslation()` and skip the runtime compiler entirely: [Pre-compile translations recipe](https://ngx-translate.org/recipes/precompile-translations/).
