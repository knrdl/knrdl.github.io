---
title: "The magic world of HTML inputs"
date: "2022-09-11"
tags: [html5, webdev]
lang: "en"
hackerNewsId: ""
---

The HTML `input` element is the principal construct when it comes to site local interactivity from user provided data on the web. Here are my 5 cents to the topic.

# Button Tag

There is a `<input type="button">`. However, it only allows unformatted text as button content. Use `<button type="button">` instead. The `type="button"` might seems redundant, but it prevents browsers from interpreting the button as `type="submit"` if there is only one button in a `<form>`.

```html
<input type="button" value="simple button">
<button type="button">
    <span style="color:orange">styled button</span>
</button>
```

<input type="button" value="simple button">
<button type="button">
    <span style="color:orange">styled button</span>
</button>

# Example values are no placeholders!

Providing the attribute `placeholder` is a good idea. But it's easy for users to mistake placeholders for actual values. For example: What is the selected limit value here?

```html
Wrong: <input type="number" placeholder="100" value="">
Correct with default: <input type="number" placeholder="limit" value="100">
Correct without default: <input type="number" placeholder="limit" value="">
```

<div>Wrong: <input type="number" placeholder="100" value=""></div>
<div>Correct with default: <input type="number" placeholder="limit" value="100"></div>
<div>Correct without default: <input type="number" placeholder="limit" value=""></div>

> Using actual values as placeholders is only a good idea, if the chosen value is always the default value. If there is no default, don't misuse an example value as a placeholder!
> {.info }

# Use the system picker

Often the first idea to create date or color pickers is to use one of the dozen available npm-modules. But these solutions offer no system integration and tend to be hacky or misfitting on mobile screens. Instead, the html5 elements provide quite an acceptable user experience on all modern devices.

```html
Date input: <input type="date" value="2022-01-01">
Time input: <input type="time" value="14:00">
Color input: <input type="color" value="#123456">
```

<div>Date input: <input type="date" value="2022-01-01"></div>
<div>Time input: <input type="time" value="03:00"></div>
<div>Color input: <input type="color" value="#123456"></div>

It's even possible to use the picker dialog without the corresponding input field:

```html

<button type="button" onclick="document.getElementById('date-input').showPicker()">
    <input type="date" style="position: absolute; visibility: hidden;" id="date-input">
    📆 Select a date
</button>
```

<button type="button" onclick="document.getElementById('date-input').showPicker()">
<input type="date" style="position: absolute; visibility: hidden;" id="date-input">
    📆 Select a date
</button>

# Autocomplete

Autocompleting text is pretty easy as well. However, if more sophisticated suggestions are required (formatted text, images, ...) then you will have to build it on your own.

```html
<input list="autocomplete-values" placeholder="Search a location">

<datalist id="autocomplete-values">
    <option value="Berlin">Berlin</option>
    <option value="Hamburg">Hamburg</option>
    <option value="Munich">Munich</option>
</datalist>
```

<input list="autocomplete-values" placeholder="Search a location">
<datalist id="autocomplete-values">
<option value="Berlin">Berlin</option>
<option value="Hamburg">Hamburg</option>
<option value="Munich">Munich</option>
</datalist>

> Old browsers might perform a startswith-search instead of a contains-search to provide completion results!
> {.warning }

Autocomplete suggestions can be updated dynamically. With Svelte it might look like this:

```svelte
<datalist id="list-id">
    {#each items as item}
        <option value="{item}">{item}</option>
    {/each}
</datalist>
```

# Take advantage of semantics

HTML is a semantic markup language. While `<input type="text">` works, it's sometimes not the best fit:

* `<input type="password" autocomplete="new-password">` allows the browser to generate and store secure passwords
* `<input type="password" autocomplete="current-password">` allows the browser to autofill a stored password
* `<input type="tel">` allows the browser to display a phone number keypad
* `<input type="search">` allows the user to define a custom keyword as additional search engine for the browser
* ...

Use labels to combine input description and control element:

```html
<label>Select a value:
    <input type="range">
</label>
```

<label>Select a value:
<input type="range">
</label>

# Input Validation

## 1. Define Constraints

```html

<form class="validate">
    <input type="email" required pattern="[^@]+@\w+\.\w+">
    <input type="url" required pattern="^https?://.+">
    <input type="text" required minlength="3" maxlength="8" pattern="\w+">
    <input type="number" required min="0" max="1000">
</form>
```

## 2. Visual Feedback

```html

<style>
    form.validate input:invalid {
        border: 2px solid red;
    }

    form.validate input:valid {
        border: 2px solid black;
    }
</style>
```

Keep in mind that controls like `<textare>` or `<select>` also use the pseudo-selectors `:valid` and `:invalid`. They can be combined with e.g. `:required` or `:focus`.

## 3. Provide custom error messages

Either via [JavaScript](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation#validating_forms_using_javascript) or via [CSS](https://css-tricks.com/snippets/css/form-validation-styling-on-input-focus/).

> Clientside form validation is an improvement to the user experience but no security measure. All user provided data must be sanitized and strictly validated serverside.
> {.warning }

# Multimedia

The file picker `<input type="file">` can be used to take photos or videos on mobile devices:

```html
<input type="file" onchange="addPhotos(this.files)" accept="image/*" multiple="multiple">
<input type="file" onchange="addVideos(this.files)" accept="video/*" multiple="multiple">
```

# Foobar

`<input type="image">` is much like `<button type="submit">`, except that it also transmits the xy coordinates where the user clicked on the image. This could be used e.g. to zoom in on a geo map *without any javascript*. Click on the image below and watch the url parameter.
<form>
<input type="image" src="https://tile.openstreetmap.org/11/1098/674.png" width="256" height="256">
</form>
&copy; OpenStreetMap contributors
