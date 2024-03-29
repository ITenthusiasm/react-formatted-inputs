# Do You Really Need React State to Format Inputs?

As I've said in another article, [it's more than possible to handle forms in React without using state](https://thomason-isaiah.medium.com/you-dont-need-all-that-react-state-in-your-forms-a2c38b8e21d5). But what about formatted inputs? Let's say I have an input that's intended to take a person's phone number (or some other meaningful numeric value). You're probably used to seeing solutions that look something like this:

```tsx
import React, { useState } from "react";

export default function MyPage() {
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

export default function MyPage() {
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

This approach works. But is it really worth it? I mean... The code is more verbose... I still have to pollute 2 `input` props... and this doesn't look very re-usable. Is all of that really worth it in order to avoid unnecessary re-renders, large amounts of state variables, and the [other problems that come with controlled inputs](https://thomason-isaiah.medium.com/you-dont-need-all-that-react-state-in-your-forms-a2c38b8e21d5)?

If that's what you're thinking, then you're asking the right questions. :&rpar; Thankfully, there is a solution that beautifully resolves this concern.

# React Actions

I've been playing around with [`Svelte`](https://svelte.dev/) for a bit recently, and I quite honestly love it. I _highly_ encourage you to try it out for yourself. One of the brilliant features that `Svelte` has is [actions](https://svelte.dev/tutorial/actions). Actions enable you to add _re-usable_ functionality to an HTML element _without_ having to create a separate component. And it's all done using plain old JS functions.

I used to think that adding re-usable functionality to HTML elements was only possible in Svelte, but it's actually still possible in React thanks to `ref`s. Check this out!

(Note: If you're unfamiliar with React refs, you should see their [documentation](https://reactjs.org/docs/refs-and-the-dom.html) before continuing.)

```tsx
// components/MyPage.tsx

import React from "react";
import actFormatted from "./actions/actFormatted";

export default function MyPage() {
  return (
    <form>
      <input ref={actFormatted(/^\d*$/)} id="some-numeric-input" type="text" />
    </form>
  );
}
```

```tsx
// actions/actFormatted.ts

function actFormatted(pattern: RegExp) {
  /** Stores the react `ref` */
  let input: HTMLInputElement | null;

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

  return function (reactRef: typeof input): void {
    if (reactRef !== null) {
      input = reactRef;
      input.pattern = pattern.toString().slice(1, -1); // Strip the leading and ending forward slashes
      input.addEventListener("beforeinput", handleBeforeInput as EventListener);
      input.addEventListener("input", handleInput as EventListener);
    } else {
      input?.removeEventListener("beforeinput", handleBeforeInput as EventListener);
      input?.removeEventListener("input", handleInput as EventListener);
      input = null;
    }
  };
}

export default actFormatted;
```

I call these... **React Actions**. (Pretty cool, right?!?)

Basically, I take advantage of the `HTMLInputElement` reference that React exposes, and I hook up all of the useful handlers that I need to get the formatting job done. Because the `ref` prop that React exposes accepts a function that acts _on_ the DOM element, I'm able to create the re-usable utility function that you see above. I'm even able to update meaningful HTML attributes, like [`pattern`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/pattern)!

Notice that — just as with Svelte Actions — we have to take responsibility for cleaning up the event listeners in our React Actions. [According to the React docs](https://react.dev/reference/react-dom/components/common#ref-callback), React will call the `ref` callback with the `HTMLElement` when it's attached to the DOM. When the element is removed from the DOM, the callback will be called with `null` instead. Thus, in our function, we're making sure to _add_ the event listeners when the React reference exists (i.e., during mounting), and _remove_ the event listeners when the `reactRef` is `null` (i.e., during unmounting).

**Important:** React will _also_ call your `ref` callback whenever you pass a _different_ `ref` callback to an element or component -- not just during DOM attachment/removal. This means that if you're using a React Action in a component that re-renders, you _might_ need to memoize it with [`useMemo`](https://react.dev/reference/react/useMemo)/[`useCallback`](https://react.dev/reference/react/useCallback). (This will not always be required.)

Sidenote: You probably noticed that this time I'm adding an `oninput` handler instead of an `onchange` handler to the input element. This is intentional, as _there is a difference between `oninput` handlers and `onchange` handlers in the regular JavaScript world_. (See [this Stackoverflow question](https://stackoverflow.com/questions/17047497/difference-between-change-and-input-event-for-an-input-element).) Most well-known frontend frameworks like Vue and Svelte respect this difference (thankfully). Unfortunately, [React does not](https://github.com/facebook/react/issues/3964). And since our function is using _raw JS_ (not React), we have to use the regular `oninput` handler instead of an `onchange` handler (which is a good thing). This article is not intended to fully explain this React oddity, but I encourage you to learn more about it soon if you aren't familiar with it. It's pretty important. (That React GitHub link I just gave you is a good start.)

# What Are the Benefits to This?

This is **game changing**! And for a few reasons, too!

**First**, it means that we don't run into the issues I mentioned [in my first article about using controlled inputs](https://thomason-isaiah.medium.com/you-dont-need-all-that-react-state-in-your-forms-a2c38b8e21d5). This means we reduce code redundancy, remove unnecessary re-renders, maintain code and skills that are _transferrable between frontend frameworks_, and more!

**Second**, we have a _re-usable_ solution to our formatting problem. Someone may say, "Couldn't we have added re-usability via hooks or components?" And the answer is yes. However, in terms of re-usability, I prefer this approach over creating a custom hook or creating a re-usable component. Regarding the former, React Actions are more flexible because they aren't bound by the [Rules of Hooks](https://legacy.reactjs.org/docs/hooks-rules.html). The latter option can get you in trouble if you want more freedom over how your inputs are styled. (Plus, if you aren't using TypeScript, then redeclaring ALL the possible `HTMLInputElement` attributes as props would be a huge bother. So much for "re-usability"). Also, both of those approaches are more framework specific. _React Actions_ remove the re-rendering problem without removing re-usability. They're the best way to go for re-usability and efficiency.

**Third**, _we unblock our event handlers_. What do I mean? Well, unlike Svelte, React doesn't allow you to [define multiples of the same event handler](https://svelte.dev/repl/91a053c1a3ed4aa3ac73b0b0518bf20e?version=3.29.4) on a JSX element. So once you take up a handler, that's it. Sure, you can _simulate_ defining multiple handlers at once by doing something like this:

```tsx
export default function MyPage() {
  function handleChange(event: React.ChangeEvent<HTMLInputElement>) {
    callback1(event);
    callback2(event);
  }

  return (
    <form>
      <input onChange={handleChange} />
    </form>
  );
}
```

But that approach is rather bothersome — especially when you only want to do something as simple as format an input. By using React Actions, _we've freed up that `onChange` prop_! (Admittedly, it trades the `onChange` prop for the `ref` prop. However, `ref` is much less commonly used. And if you really need to use the `ref` more than once, you can get around the problem by using a similar approach to the one shown above.)

**Fourth** this approach is compatible with state variables! Consider this:

```tsx
import React, { useState } from "react";
import actFormatted from "./actions/actFormatted";

export default function MyPage() {
  const [value, setValue] = useState("");

  function handleChange(event: React.ChangeEvent<HTMLInputElement>) {
    setValue(event.target.value);
  }

  return (
    <form>
      <input
        ref={actFormatted(/^\d*$/)}
        id="some-numeric-input"
        type="text"
        value={value}
        onChange={handleChange}
      />
    </form>
  );
}
```

This situation acts almost _exactly the same_ as if we were just _controlling_ a regular input. The difference? Our `handleChange` event handler will _only see the formatted value_ whenever the value changes. This means you can setup an event handler that's only intended to interact with the formatted value. And you can do this _without_ creating an intermediate "re-usable component".

# "But Mutations!!!"

Since this is a React article, I imagine there are a few people who might complain about how this approach includes mutations (_not_ on state... just on `event.target`). But honestly, after playing around with some other frameworks, I've learned that there are times to mutate, and there are times not to mutate. It's better to learn both and to master the different scenarios than it is to impose standards impractically and make code more difficult to handle. There's a time and place for everything...

# And the Possibilities Don't Stop with Inputs...

You can create whatever kind of React Action you need to get the job done for your inputs. But you can go even further beyond! For instance, have you ever had to make HTMLElements act like buttons? If you've tried to do this with components, you've probably had to write something painful like this:

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

What's undesirable with this "re-usable component" approach is that we're forced to create a prop that represents the type of element to use (whether we default its value or not). I also found that making a clean, re-usable, flexible TS interface was difficult to do without running into problems — hence the use of `any`. (There might be a solution that works without `any`, but it wasn't worth excavating for. Try tinkering with the types and you'll see what I mean.)

We can make things _much_ simpler with React Actions:

```tsx
import React from "react";
import actFormatted from "./actions/actFormatted";
import actBtnLike from "./actions/actBtnLike";

export default function MyPage() {
  return (
    <form>
      <input ref={actFormatted(/^\d*$/)} id="some-numeric-input" type="text" />

      <label ref={actBtnLike()} htmlFor="file-upload">
        Upload File
      </label>
      <input id="file-upload" type="file" style={{ display: "none" }} />
    </form>
  );
}
```

```tsx
// actions/actBtnLike.ts

// Place this on the outside so that we don't have to define it every time an element mounts
function handleKeydown(event: KeyboardEvent & { currentTarget: HTMLElement }): void {
  if (event.key === "Enter" || event.key === " ") {
    event.preventDefault();
    event.currentTarget.click();
  }
}

/** Makes an element focusable and enables it to receive `keydown` events as if it were a `button`. */
function actBtnLike() {
  let element: HTMLElement | null;

  return function (reactRef: typeof element): void {
    if (reactRef !== null) {
      element = reactRef;
      element.tabIndex = 0;
      element.addEventListener("keydown", handleKeydown as EventListener);
    } else {
      element?.removeEventListener("keydown", handleKeydown as EventListener);
      element = null;
    }
  };
}

export default actBtnLike;
```

By adding a JSDoc comment, we can add some IntelliSense to our React Action so that new developers know what this tool is doing! And by placing `handleKeydown` on the outside of our action, we guarantee that the function is only defined once. (This will not always be possible, depending on how complex your React Action is. But this is still much better than having to create an _unnecessary_ component that could potentially cause _unnecessary_ re-renders.)

This is only the beginning! I encourage everyone who reads this article to explore the new possibilities for their React applications with this approach!

# Don't Forget Your Tests!

Before wrapping up, I just wanted to make sure it was clear that actions _are_ testable too!

```tsx
// actions/__tests__/actBtnLike.test.tsx

import React from "react";
import "@testing-library/jest-dom/extend-expect";
import { render } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import actBtnLike from "../actBtnLike";

describe("Act Button-Like Action", () => {
  it("Causes an element to behave like a button for keyboard events", async () => {
    const handleClick = jest.fn((event: React.MouseEvent<HTMLLabelElement, MouseEvent>) => {
      console.log("In a real app, the file navigator would be opened.");
    });

    const { getByText } = render(
      <form>
        <label ref={actBtnLike()} htmlFor="file-upload" onClick={handleClick}>
          Upload File
        </label>
        <input id="file-upload" type="file" style={{ display: "none" }} />
      </form>
    );

    // Shift focus to label element, and activate it with keyboard actions
    const label = getByText(/upload file/i);
    await userEvent.tab();
    expect(label).toHaveFocus();

    await userEvent.keyboard("{Enter}");
    expect(handleClick).toHaveBeenCalledTimes(1);

    await userEvent.keyboard(" ");
    expect(handleClick).toHaveBeenCalledTimes(2);
  });
});
```

```tsx
// actions/__tests__/actFormatted.test.tsx

import React from "react";
import "@testing-library/jest-dom/extend-expect";
import { render } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import actFormatted from "../actFormatted";

describe("Act Formatted Action", () => {
  it("Enforces the specified pattern for an `input`", async () => {
    const numbersOnlyRegex = /^\d*$/;
    const { getByLabelText } = render(
      <form>
        <label htmlFor="number-input">Numeric Input</label>
        <input id="number-input" ref={actFormatted(numbersOnlyRegex)} />
      </form>
    );

    const numericInput = getByLabelText(/numeric input/i) as HTMLInputElement;
    await userEvent.type(numericInput, "abc1def2ghi3");
    expect(numericInput).toHaveValue("123");
  });
});
```

Yep. Pretty straightforward. Note that — as is the case for all kinds of testing — your testing capabilities are limited to your testing tools. For instance, version 14 (or higher) of [User Event Testing Library](https://testing-library.com/docs/user-event/intro) is needed to support tests for anything that relies on `beforeinput`. (When using this version of the package for your tests, remember to `await` all of your calls to the `userEvent` functions.) If you're determined to use an earlier version of the package, you'll have to run with [Cypress Testing Library](https://testing-library.com/docs/cypress-testing-library/intro/) instead.

---

And that's a wrap! Hope this was helpful! Let me know with a [shoutout on Twitter](https://twitter.com/ITEnthusiasm) maybe? 😄

I want to give a **_HUUUUUUUUUUGE_** thanks to Svelte! That is, to everyone who works so hard on that project! It really is a great framework worth checking out if you haven't done so already. I _definitely_ would not have discovered this technique if it wasn't for them. And I want to give a special, specific shoutout to [@kevmodrome](https://twitter.com/kevmodrome) from [Svelte Society](https://sveltesociety.dev/). He helped me out with a problem related to formatting inputs in Svelte land. He has a [great article on actions](https://svelte.school/tutorials/introduction-to-actions) if you're interested in learning more about what you can do with them. It's technically for Svelte, but you can still apply what's there to "React Actions". :&rpar;

I want to extend another **enormous** thanks to [@willwill96](https://github.com/willwill96)! He caught an implementation bug in an earlier version of this article. 😬

Almost finally, I want to extend another big thanks to my current employer (at the time of this writing), [MojoTech](https://www.mojotech.com/). ("Hi Mom!") Something unique about MojoTech is that they give their engineers time each week to explore new things in the software world and expand their skills. Typically, I learn _most_ and _fastest_ on side projects (when it comes to software, at least). If it wasn't for them, I probably wouldn't have been able to explore Svelte, which means I wouldn't have fallen in love with the framework and learned about actions, which means this article never would have existed. 😩

And last but farthest from least, I want to thank Jesus Christ. He's the One who ultimately enables me to do anything meaningful that I do.

Give honor where honor is due, right? :smile:
