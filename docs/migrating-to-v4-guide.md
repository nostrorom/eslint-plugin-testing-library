# Guide: migrating to v4

Previous versions of `eslint-plugin-testing-library` weren't checking common things consistently: Testing Library imports, renamed methods, wrappers around Testing Library methods, etc.
One of the most important changes of `eslint-plugin-testing-library` v4 is the new detection mechanism implemented to be shared across all the rules, so each one of them has been rewritten to detect and report Testing Library usages consistently and correctly from a core module.

## Overview

- [Aggressive Reporting](#aggressive-reporting) opted-in to avoid silencing possible errors
- 7 new rules
  - `no-container`
  - `no-node-access`
  - `no-promise-in-fire-event`
  - `no-wait-for-multiple-assertions`
  - `no-wait-for-side-effects`
  - `prefer-user-event`
  - `render-result-naming-convention`
- Shareable Configs updated
  - `recommended` renamed to `dom`
  - list of rules enabled has changed
- Some rules option removed in favor of new Shared Settings + Aggressive Reporting
- More consistent and flexible core rules detection
- Tons of errors and small issues fixed
- Dependencies updates

## Breaking Changes

### Dependencies

- Min ESLint version required is `7.5`
- Min Node version required didn't change (`10.12`)

Please make sure you have Node and ESLint installed satisfying these required versions.

### New errors reported

Since v4 also fixes a lot of issues and detect invalid usages in a more consistent way, you might find new errors reported in your codebase. Just be aware of this when migrating to v4.

### `recommended` Shareable Config has been renamed

If you were using `recommended` Shareable Config, it has been renamed to `dom` so you'll need to update it in your ESLint config file:

```diff
{
  ...
- "extends": ["plugin:testing-library/recommended"]
+ "extends": ["plugin:testing-library/dom"]
}
```

This Shareable Config has been renamed to clarify there is no _recommended_ config by default, so it depends on which Testing Library package you are using: DOM, Angular, React, or Vue (for now).

### Shareable Configs updated

Shareable Configs have been updated with:

- `dom`
  - `no-promise-in-fire-event` enabled as "error"
  - `no-wait-for-empty-callback` enabled as "error"
  - `prefer-screen-queries` enabled as "error"
- `angular`
  - `no-container` enabled as "error"
  - `no-debug` changed from "warning" to "error"
  - `no-node-access` enabled as "error"
  - `no-promise-in-fire-event` enabled as "error"
  - `no-wait-for-empty-callback` enabled as "error"
  - `prefer-screen-queries` enabled as "error"
  - `render-result-naming-convention` enabled as "error"
- `react`
  - `no-container` enabled as "error"
  - `no-debug` changed from "warning" to "error"
  - `no-node-access` enabled as "error"
  - `no-promise-in-fire-event` enabled as "error"
  - `no-wait-for-empty-callback` enabled as "error"
  - `prefer-screen-queries` enabled as "error"
  - `render-result-naming-convention` enabled as "error"
- `vue`
  - `no-container` enabled as "error"
  - `no-debug` changed from "warning" to "error"
  - `no-node-access` enabled as "error"
  - `no-promise-in-fire-event` enabled as "error"
  - `no-wait-for-empty-callback` enabled as "error"
  - `prefer-screen-queries` enabled as "error"
  - `render-result-naming-convention` enabled as "error"

### `customQueryNames` rules option has been removed

Until now, those rules reporting errors related to Testing Library queries needed an option called `customQueryNames` so you could specify which extra queries you'd like to report apart from built-in ones. This option has been removed in favor of reporting every method matching Testing Library queries pattern. The only thing you need to do is removing `customQueryNames` from your rules config if any. You can read more about it in corresponding [Aggressive Reporting - Queries](#queries) section.

### `renderFunctions` rules option has been removed

Until now, those rules reporting errors related to Testing Library `render` needed an option called `renderFunctions` so you could specify which extra functions from your codebase should be assumed as extra `render` methods apart from built-in one. This option has been removed in favor of reporting every method which contains `*render*` on its name. The only thing you need to do is removing `renderFunctions` from your rules config if any. You can read more about it in corresponding [Aggressive Reporting - Render](#renders) section, and available config in [Shared Settings](#shared-settings) section.

## Aggressive Reporting

So what is this Aggressive Reporting introduced on v4? Until v3, `eslint-plugin-testing-library` had assumed that all Testing Libraries utils would be imported from some `@testing-library/*` or `*-testing-library` package. However, this is not always true since:

- users can [add their own Custom Render](https://testing-library.com/docs/react-testing-library/setup/#custom-render) methods, so it can be named other than `render`.
- users can [re-export Testing Library utils from a custom module](https://testing-library.com/docs/react-testing-library/setup/#configuring-jest-with-test-utils), so they won't be imported from a Testing Library package but a custom one.
- users can [add their own Custom Queries](https://testing-library.com/docs/react-testing-library/setup/#add-custom-queries), so it's possible to use other queries than built-in ones.

These customization mechanisms make impossible for `eslint-plugin-testing-library` to figure out if some utils are related to Testing Library or not. Here you have some examples illustrating it:

```javascript
import { render, screen } from '@testing-library/react';
// ...

// ✅ this render has to be reported since it's named `render`
// and it's imported from @testing-library/* package
const wrapper = render(<Component />);

// ✅ this query has to be reported since it's named after a built-in query
// and it's imported from @testing-library/* package
const el = screen.findByRole('button');
```

```javascript
// importing from Custom Module
import { renderWithRedux, findByIcon } from 'test-utils';
// ...

// ❓ we don't know if this render has to be reported since it's NOT named `render`
// and it's NOT imported from @testing-library/* package
const wrapper = renderWithRedux(<Component />);

// ❓ we don't know if this query has to be reported since it's NOT named after a built-in query
// and it's NOT imported from @testing-library/* package
const el = findByIcon('profile');
```

How can the `eslint-plugin-testing-library` be aware of this? Until v3, the plugin offered some options to indicate some of these custom things, so the plugin would check them when reporting usages. This can lead to false negatives though since the users might not be aware of the necessity of indicating such custom utils or just forget about doing so.

Instead, in `eslint-plugin-testing-library` v4 we have opted-in a more **aggressive reporting** mechanism which, by default, will assume any method named following the same patterns as Testing Library utils has to be reported too:

```javascript
// importing from Custom Module
import { renderWithRedux, findByIcon } from 'test-utils';
// ...

// ✅ this render has to be reported since its name contains "*render*"
// and it doesn't matter where it's imported from
const wrapper = renderWithRedux(<Component />);

// ✅ this render has to be reported since its name starts by "findBy*"
// and it doesn't matter where it's imported from
const el = findByIcon('profile');
```

There are 3 behaviors then that can be aggressively reported: imports, renders, and queries. This new Aggressive Reporting mechanism will just work fine out of the box and won't create false positives for most of the users. However, it's possible to do some tweaks to disable some of these behaviors using the new [Shared Settings](#shared-settings). We recommend you to keep reading this section to know more about these Aggressive Reporting behaviors and then check the Shared Settings if you think you'd still need it for some particular reason.

_You can find the motivation behind this behavior on [this issue comment](https://github.com/testing-library/eslint-plugin-testing-library/issues/222#issuecomment-679592434)._

### Imports

By default, `eslint-plugin-testing-library` v4 won't check from which module are the utils imported. This means it doesn't matter if you are importing the utils from `@testing-library/*`, `test-utils` or `whatever`.

There is a new Shared Setting to restrict this scope though: [`utils-module`](#testing-libraryutils-module). By using this setting, only utils imported from `@testing-library/*` packages, or the custom one indicated in this setting would be reported.

### Renders

By default, `eslint-plugin-testing-library` v4 will assume that all methods which names contain "render" should be reported. This means it doesn't matter if you are rendering your elements for testing using `render`, `customRender` or `renderWithRedux`.

There is a new Shared Setting to restrict this scope though: [`custom-renders`](#testing-librarycustom-renders). By using this setting, only methods strictly named `render` or as one of the indicated Custom Renders would be reported.

### Queries

`eslint-plugin-testing-library` v4 will assume that all methods named following the pattern `get(All)By*`, `query(All)By*`, or `find(All)By*` are queries to be reported. This means it doesn't matter if you are using a built-in query (`getByText`), or a custom one (`getByIcon`): if it matches this pattern, it will be assumed as a potential query to be reported.

There is no way to restrict this behavior for now.

## Shared Settings

ESLint has a setting feature which allows configuring data that must be shared across all its rules: [Shared Settings](https://eslint.org/docs/user-guide/configuring/configuration-files#adding-shared-settings). Since `eslint-plugin-testing-library` v4 we are using this Shared Settings to config global things for the plugin.

To avoid collision with settings from other ESLint plugins, all the properties for this one are prefixed with `testing-library/`.

⚠️ **Please be aware of using these settings will disable part of [Aggressive Reporting](#aggressive-reporting).**

### `testing-library/utils-module`

The name of your custom utility file from where you re-export everything from Testing Library package. Relates to [Aggressive Reporting - Imports](#imports).

```json
// .eslintrc
{
  "settings": {
    "testing-library/utils-module": "my-custom-test-utility-file"
  }
}
```

Enabling this setting, you'll restrict the errors reported by the plugin to only those utils being imported from this custom utility file, or some `@testing-library/*` package. The previous setting example would cause:

```javascript
import { waitFor } from '@testing-library/react';

test('testing-library/utils-module setting example', () => {
  // ✅ this would be reported since this invalid usage of an util
  // is imported from `@testing-library/*` package
  waitFor(/* some invalid usage to be reported */);
});
```

```javascript
import { waitFor } from '../my-custom-test-utility-file';

test('testing-library/utils-module setting example', () => {
  // ✅ this would be reported since this invalid usage of an util
  // is imported from specified custom utility file.
  waitFor(/* some invalid usage to be reported */);
});
```

```javascript
import { waitFor } from '../somewhere-else';

test('testing-library/utils-module setting example', () => {
  // ❌ this would NOT be reported since this invalid usage of an util
  // is NOT imported from either `@testing-library/*` package or specified custom utility file.
  waitFor(/* some invalid usage to be reported */);
});
```

### `testing-library/custom-renders`

A list of function names that are valid as Testing Library custom renders. Relates to [Aggressive Reporting - Renders](#renders)

```json
// .eslintrc
{
  "settings": {
    "testing-library/custom-renders": ["display", "renderWithProviders"]
  }
}
```

Enabling this setting, you'll restrict the errors reported by the plugin related to `render` somehow to only those functions sharing a name with one of the elements of that list, or built-in `render`. The previous setting example would cause:

```javascript
import {
  render,
  display,
  renderWithProviders,
  renderWithRedux,
} from 'test-utils';
import Component from 'somewhere';

const setupA = () => renderWithProviders(<Component />);
const setupB = () => renderWithRedux(<Component />);

test('testing-library/custom-renders setting example', () => {
  // ✅ this would be reported since `render` is a built-in Testing Library util
  const invalidUsage = render(<Component />);

  // ✅ this would be reported since `display` has been set as `custom-render`
  const invalidUsage = display(<Component />);

  // ✅ this would be reported since `renderWithProviders` has been set as `custom-render`
  const invalidUsage = renderWithProviders(<Component />);

  // ❌ this would NOT be reported since `renderWithRedux` isn't a `custom-render` or built-in one
  const invalidUsage = renderWithRedux(<Component />);

  // ✅ this would be reported since it wraps `renderWithProviders`,
  // which has been set as `custom-render`
  const invalidUsage = setupA(<Component />);

  // ❌ this would NOT be reported since it wraps `renderWithRedux`,
  // which isn't a `custom-render` or built-in one
  const invalidUsage = setupB(<Component />);
});
```