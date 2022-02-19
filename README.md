# Do You Really Need React State to Format Inputs?

As I've said in another article, [it's more than possible to handle forms in React without using state](https://github.com/ITenthusiasm/react-uncontrolled-inputs). But what about formatted inputs? Let's say I have an input that's intended to take a person's phone number (or some other meaningful numeric value). You're probably used to seeing solutions that look something like this:

```tsx
import React, { useState } from "react";

function MyPage() {
  const [myNumericValue, setMyNumericValue] = useState("");

  function handleChange(event: Event & React.ChangeEvent<HTMLInputElement>) {
    const { value } = event.target;
    if (/^\d*$/.test(value)) setMyNumericValue(value);
  }

  return (
    <form>
      <input id="some-numeric-input" type="text" value={myNumericValue} onChange={handleChange} />
    </form>
  );
}
```

(If you want, you can follow along in a codesandbox as you read this article. [This codesandbox](https://codesandbox.io/s/react-formatted-inputs-starting-code-21hy51) starts you off with the code you see above.)

Okay... This works... But this brings us back to the problem of _needing_ state variables to handle our inputs. The problem is worse if we have _several_ formatted inputs. If you're like me and you don't want to inherit all of the disadvantages of relying on state for your forms, there is another way...

```tsx
import React from "react";

function MyPage() {
  /** Tracks the last valid value of the input. */
  let lastValidValue: string;

  /** Helps ensure that the last valid value is maintained in the input. */
  function handleBeforeInput(event: React.FormEvent<HTMLInputElement>) {
    lastValidValue = (event.target as HTMLInputElement).value;
  }

  /** Allows the user to input correctly-formatted values. Blocks invalid inputs. */
  function handleChange(event: React.ChangeEvent<HTMLInputElement>) {
    const { value, selectionStart } = event.target;

    // Invalid input value
    if (!/^\d*$/.test(value)) {
      event.target.value = lastValidValue;

      // Keep the cursor in the right location if the user input was invalid
      const cursorPlace = (selectionStart as number) - (value.length - event.target.value.length);
      requestAnimationFrame(() => event.target.setSelectionRange(cursorPlace, cursorPlace));
      return;
    }

    // Input value is valid. Synchronize `lastValidValue`
    lastValidValue = value;
  }

  return (
    <form>
      <input
        id="some-numeric-input"
        type="text"
        onBeforeInput={handleBeforeInput}
        onChange={handleChange}
      />
    </form>
  );
}
```

(If you're unfamiliar with the `beforeinput` event, I encourage you to check out the [MDN docs](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/beforeinput_event). But essentially, `beforeinput` gives you access to an input's value _before_ any changes have occurred. This is incredibly useful for keeping the `input`'s value valid.)

This approach works. But is it really worth it? I mean... The code is more verbose... I still have to pollute 2 `input` props... and this doesn't look very re-usable. Is all of that really worth it in order to avoid unnecessary re-renders, large amounts of state variables, and the [other problems that come with controlled inputs](https://github.com/ITenthusiasm/react-uncontrolled-inputs)?

If that's what you're thinking, then you're asking the right questions. :) Thankfully, there is a solution that beautifully resolves this concern.

## React Actions

I've been playing around with [`Svelte`](https://svelte.dev/) for a bit recently, and I quite honestly love it. I _highly_ encourage you to try it out for yourself. One of the brilliant features that `Svelte` has is [actions](https://svelte.dev/tutorial/actions). Actions enable you to add _re-usable_ functionality to an HTML element _without_ having to create a separate component. And it's all done using plain old JS functions. I first learned about actions when [Kevin](https://twitter.com/kevmodrome) from [Svelte Society](https://sveltesociety.dev/) helped me out with a problem on formatting inputs. He has a great article on actions [here](https://svelte.school/tutorials/introduction-to-actions). But you're here for React, right?

I used to think that adding re-usable functionality to HTML elements was only possible in Svelte, but it's actually still possible in React thanks to `ref`s. Check this out!

(Note: If you're unfamiliar with React refs, you should see their [documentation](https://reactjs.org/docs/refs-and-the-dom.html) before continuing.)

```tsx
// components/MyPage.tsx
import React from "react";
import useFormat from "actions/useFormat";

function MyPage() {
  return (
    <form>
      <input ref={useFormat(/^\d*$/)} id="some-numeric-input" type="text" />
    </form>
  );
}
```

```ts
// actions/useFormat.ts
function useFormat(pattern: RegExp) {
  /** Tracks the last valid value of the input. */
  let lastValidValue: string;

  /** Helps ensure that the last valid value is maintained in the input. */
  function handleBeforeInput(event: Event & { target: HTMLInputElement }): void {
    lastValidValue = event.target.value;
  }

  /** Allows the user to input correctly-formatted values. Blocks invalid inputs. */
  function handleInput(event: Event & { target: HTMLInputElement }): void {
    const { value, selectionStart } = event.target;

    if (!/^\d*$/.test(value)) {
      event.target.value = lastValidValue;
      const cursorPlace = (selectionStart as number) - (value.length - event.target.value.length);
      requestAnimationFrame(() => event.target.setSelectionRange(cursorPlace, cursorPlace));
      return;
    }

    lastValidValue = value;
  }

  return function (input: HTMLInputElement | null): void {
    if (!input) return;

    input.pattern = pattern.toString().slice(1, -1); // Strip the leading and ending forward slashes
    input.addEventListener("beforeinput", handleBeforeInput as EventListener);
    input.addEventListener("input", handleInput as EventListener);
  };
}

export default useFormat;
```

I call these... **React Actions**. (Did that give you a strong reaction? :smirk:)

Basically, I take advantage of the HTMLInputElement reference that React exposes, and I hook up all the useful handlers that I need to get the formatting job done. Because the `ref` prop that React exposes accepts a function that acts _on_ the DOM element, I'm actually able to create this re-usable utility function that you see above. I'm even able to update meaningful HTML attributes, such as [pattern](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/pattern)!

Because the `input` reference (`ref`) can be `null` at the very start of the `input`'s life, I do need a single null check. But that isn't a big deal. Everything gets registered correctly after the `input` is properly loaded into the DOM.

Note: You probably noticed that this time I'm adding an `oninput` handler instead of an `onchange` handler to the input element. This is intentional, as _there is a difference between `oninput` handlers and `onchange` handlers in the regular JavaScript world_. See [this Stackoverflow question](https://stackoverflow.com/questions/17047497/difference-between-change-and-input-event-for-an-input-element). Most well-known frontend frameworks like Vue and Svelte respect this difference (thankfully). Unfortunately, [React does not](https://github.com/facebook/react/issues/3964). And since our function is using _raw JS_ (not React), we have to use the regular `oninput` handler instead of an `onchange` handler (which is a good thing). This article is not intended to fully explain this React oddity, but I encourage you to learn more about it soon if you aren't familiar with it. It's pretty important. (That React GitHub link I just gave you is a good start.)

## What Are the Benefits to This?

This is **game changing**! And for a few reasons, too!

**First**, it means that we don't run into the issues I mentioned [in my first article about using controlled inputs](https://github.com/ITenthusiasm/react-uncontrolled-inputs). This means we reduce code redundancy, remove unnecessary re-renders, maintain code and skills that are _transferrable between frontend frameworks_, and more!

**Second**, we have a _re-usable_ solution to our formatting problem. Someone may say, "Couldn't we have added re-usability via components?" And the answer is yes. However, in terms of re-usability, I prefer this approach over creating a custom hook or creating a re-usable component. Regarding the former, it just seems odd to use hooks for something so simple. The latter option can get you in trouble if you want more freedom over how your inputs are styled. (Plus, if you aren't using TypeScript, then redeclaring ALL the possible HTMLInputElement attributes as props would be a huge bother.) So much for "re-usability". Also, both of those approaches are very framework specific, and they still leave you with unnecessary re-renders in one way or another. _React Actions_ remove the re-rendering problem without removing re-usability. They're the best way to go for re-usability and efficiency.

**Third**, _we unblock our event handlers_. What do I mean? Well, unlike Svelte, React doesn't allow you to [define multiples of the same event handler](https://svelte.dev/repl/91a053c1a3ed4aa3ac73b0b0518bf20e?version=3.29.4) on a JSX element. So once you take up a handler, that's it. Sure, you can _simulate_ defining multiple handlers at once by doing something like this:

```tsx
MyPage() {
  function handleChange(event: React.ChangeEvent<HTMLInputElement>) {
    callback1(event);
    callback2(event);
  }

  return (
    <form>
      <input onChange={handleChange} />
    </form>
  )
}
```

But... that approach is rather bothersome -- especially when you only want to do something as simple as format an input. By using React Actions, we've _freed up that `onChange` prop_! (Admittedly, it trades the `onChange` prop for the `ref` prop. However, `ref` is much less commonly used. And if you really need to use the `ref` more than once, you can get around the problem by using a similar approach to the one showed above.)

**Fourth** this approach is compatible with state variables! Consider this:

```tsx
import React, { useState } from "react";
import useFormat from "actions/useFormat";

function MyPage() {
  const [value, setValue] = useState("");

  function handleChange(event: React.ChangeEvent<HTMLInputElement>) {
    setValue(event.target.value);
  }

  return (
    <form>
      <input
        ref={useFormat(/^\d*$/)}
        id="some-numeric-input"
        type="text"
        value={value}
        onChange={handleChange}
      />
    </form>
  );
}
```

This situation acts almost _exactly the same_ as if we were just using a regular input. The difference? Our `handleChange` event handler will _only_ see the _formatted_ value whenever the value changes. This means you can setup an event handler that's only intended to interact with the formatted value. And you can do this _without_ needing an intermediary "re-usable component".

## "But Mutations!!!"

Since this is a React article, I imagine there are a few people who might complain about how this approach includes mutations (_not_ on state... just on `event.target`). But honestly, after playing around with some other frameworks, I've learned that there are times to mutate, and there are times not to mutate. Better to learn both and master the different situations than to impose standards impractically and make code more difficult to handle. There's a time and place for everything...

## And the Possibilities Don't Stop with Inputs...

You can create whatever kind of React Action you need to get the job done for your inputs. But you can go even further beyond! For instance, have you ever had to make HTML Elements act as if they're buttons? Please don't tell me you're still under the slavery of using "re-usable components":

```tsx
interface ButtonLikeProps<T extends keyof HTMLElementTagNameMap>
  extends React.HTMLAttributes<HTMLElementTagNameMap[T]> {
  as: T;
}

function ButtonLike<T extends keyof HTMLElementTagNameMap>({
  children,
  as,
  ...attrs
}: ButtonLikeProps<T>) {
  const Element = as as any;

  function handleKeydown(event: React.KeyboardEvent<HTMLElementTagNameMap[T]>): void {
    if (["Enter", " "].includes(event.key)) {
      event.preventDefault();
      event.currentTarget.click();
    }
  }

  return (
    <Element role="button" tabIndex={0} onKeyDown={handleKeydown} {...attrs}>
      {children}
    </Element>
  );
}
```

Nope. Don't like it. It's a little lame that with the "re-usable component" approach, we're forced to create a prop that represents the type of element to use (whether we default its value or not). I also found that making a clean, re-usable, flexible TS interface was difficult to do without running into problems, hence the one disugsting use of `any`. (There might be a solution that works without `any`, but it wasn't worth excavating for. Try tinkering with the types and you'll see what I mean.)

We can make things _much_ simpler with React Actions:

```tsx
import useBtnLike from "actions/btnLike";

function MyPage() {
  // ...

  return (
    <form>
      {/* ... */}

      <label ref={useBtnLike} htmlFor="file-upload">
        Upload File
      </label>
      <input id="file-upload" type="file" style={{ display: "none" }} />
    </form>
  );
}
```

```ts
// actions/useBtnLike.ts

// Place this on the outside so that we don't have to define it every time an element mounts
function handleKeydown(event: KeyboardEvent & { currentTarget: HTMLElement }): void {
  if (event.key === "Enter" || event.key === " ") {
    event.preventDefault();
    event.currentTarget.click();
  }
}

/** Makes an element focusable and enables it to receive `keydown` events as if it were a `button`. */
function useBtnLike(element: HTMLElement | null): void {
  // Null check
  if (!element) return;

  element.tabIndex = 0;
  element.addEventListener("keydown", handleKeydown as EventListener);
}

export default useBtnLike;
```

By adding a JSDoc comment, we can add some IntelliSense to our React Action so that new developers know what this guy is doing! And by placing `handleKeydown` on the outside of our action, we guarantee that the function is only defined once. (This will not always be possible, depending on how complex your React Action is. But this is still much better than having to create an _unnecessary_ component that could potentially perform _unnecessary_ re-renders.)

This is only the beginning! I encourage everyone who reads this article to explore the new possibilities for their React applications with this approach!

## Don't Forget Your Tests!

Before wrapping up, I just wanted to make sure it was clear that actions _are_ testable too!

```tsx
// actions/__tests__/useBtnLike.test.tsx

import React from "react";
import "@testing-library/jest-dom/extend-expect";
import { render } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import useBtnLike from "../useBtnLike";

describe("Use Button-Like Action", () => {
  it("Causes an element to behave like a button for keyboard events", () => {
    const handleClick = jest.fn((event: React.MouseEvent<HTMLLabelElement, MouseEvent>) => {
      console.log("In a real app, the file navigator would be opened.");
    });

    const { getByText } = render(
      <form>
        <label ref={useBtnLike} htmlFor="file-upload" onClick={handleClick}>
          Upload File
        </label>
        <input id="file-upload" type="file" style={{ display: "none" }} />
      </form>
    );

    // Shift focus to label element, and activate it with keyboard actions
    const label = getByText(/upload file/i);
    userEvent.tab(); // Proves the element is focusable
    userEvent.keyboard("{Enter}");
    expect(handleClick).toHaveBeenCalledTimes(1);

    userEvent.keyboard(" ");
    expect(handleClick).toHaveBeenCalledTimes(2);
  });
});
```

```tsx
// actions/__tests__/useFormat.test.tsx

/*
 * NOTE: Requires `@testing-library/user-event>=14.0.0-beta.11`. Before this version,
 * User Event Testing Library did not support testing anything that relied on
 * `beforeinput`. v14 also makes the `userEvent` functions ASYNCHRONOUS. I highly recommend
 * using v14. If you choose not to, you'll need Cypress Testing Library to test this action.
 */
import React from "react";
import "@testing-library/jest-dom/extend-expect";
import { render } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import useFormat from "../useFormat";

describe("Use Format Action", () => {
  it("Enforces the specified pattern for an `input`", async () => {
    const numbersOnlyRegex = /^\d*$/;
    const { getByLabelText } = render(
      <form>
        <label htmlFor="number-input">Numeric Input</label>
        <input id="number-input" ref={useFormat(numbersOnlyRegex)} />
      </form>
    );

    const numericInput = getByLabelText(/numeric input/i) as HTMLInputElement;
    await userEvent.type(numericInput, "abc1def2ghi3");
    expect(numericInput).toHaveValue("123");
  });
});
```

(If you're following along with these examples in the codesandbox, please note that the test for `useBtnLike` uses v13 of `userEvent`, whereas the test for `useFormat` uses v14. If you're using v14 for both tests, remember to `await` all of your calls to the `userEvent` functions.)

Yep. Pretty straightforward. Note that -- as is the case for all kinds of testing -- your testing capabilities are limited to your testing tools. For instance, as noted above, version 14 of [User Event Testing Library](https://testing-library.com/docs/user-event/intro) is needed to support tests for anything that relies on `beforeinput`. If you're determined to use an earlier version of the package, you'll have to run with [Cypress Testing Library](https://testing-library.com/docs/cypress-testing-library/intro/) to handle these kinds of edge cases.

---

And that's a wrap! Hope this was helpful! Let me know with a clap or a [shoutout on Twitter](https://twitter.com/ITEnthusiasm) maybe? :smile:

I want to give a **_HUUUUUUUUUUGE_** thanks to Svelte! That is, to everyone who works so hard on that project! It really is a great framework worth checking out if you haven't done so already. I _definitely_ would not have discovered this technique if it wasn't for them. And I want to give a special second shoutout to [@kevmodrome](https://twitter.com/kevmodrome) again for the help I mentioned earlier.

Finally, I want to extend another big thanks to my current employer (at the time of this writing), [MojoTech](https://www.mojotech.com/). ("Hi Mom!") Something unique about MojoTech is that they give their engineers time each week to explore new things in the software world and expand their skills. Typically, I learn _most_ and _fastest_ on side projects (when it comes to software, at least). If it wasn't for them, I probably wouldn't have been able to explore Svelte, which means I wouldn't have fallen in love with the framework and learned about actions, which means this article never would have existed. :weary:
