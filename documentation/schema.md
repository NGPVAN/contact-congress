# Schema

This document describes how we're currently translating the process of filling out a web form into [YAML](http://www.yaml.org/), a human-readable data serialization format.

This is a working document in that the schema is evloving. While we've based this format heavily on the [Capybara](http://jnicklas.github.io/capybara/) testing framework for ruby, we haven't gotten into the full swing of using the format yet, so things are subject to change **([and open to suggestions!](https://github.com/unitedstates/congress-contact/issues))**

You can [jump to the examples](#examples) if you just want a quick reference.

---

## Top-level attributes

The top level of a member's contact schema includes only two fields: `bioguide` and `contact_form`.
`bioguide` is the legislator's assigned [Bioguide ID](http://bioguide.congress.gov) which can be used to connect this data to other data sources in the [unitedstates project](https://github.com/unitedstates).
`contact_form` is a nested hash of the pertinent details of successfully filling out and validating receipt of the member's contact form.

## Contact form fields

**method**

The HTTP method used to submit the form in all caps, most likely `GET` or `POST`

**action**

The URL the form should submit to. An empty string ('') can be used to represent the URL the form is located at, but otherwise, this should be an absolute URL like `http://github.com/unitedstates/congress-contact`, rather than `/unitedstates/congress-contact`, even if that is what appears in the form.

**usePhantomJsLogging**

This should be set to true when the contact form uses a browser alert to indicate a successful submission.

**steps**

A list of the steps that make up a successful submission of the form. Steps are
a subset of [Capybara methods](http://rubydoc.info/github/jnicklas/capybara/master#Interacting_with_forms), one of:

- [`visit`](#visit): The act of navigating to a given url.
- [`iframe`](#visit): Locates an iframe to switch focus to.
- [`find`](#find): Locating a selector on the page, an indication that no further
    steps should be executed until the selector is present and visible.
- [`fill_in`](#fill_in): Entering text into a text `input` or `textarea`.
- [`select`](#select): Choosing a value from a `select` list.
- [`wait`](#wait): Waits a time in seconds
- [`check`](#checkuncheckchoose): Ticking a checkbox `input`.
- [`uncheck`](#checkuncheckchoose) The opposite of `check`.
- [`choose`](#checkuncheckchoose) Ticking a specific item in a set of radio
    buttons.
- [`click_on`](#click_on) Clicking a link or `button`, most likely to submit a
    `form`.
- [`remove`](#remove) Removes DOM element
- [`picker`](#picker) Uses provided key-selector mapping to select element based on provided value
- [`recaptcha`](#recaptcha) Handles reCAPTCHA V2
- [`javascript`](#javascript) Allows running javascript commands

**success**

A basic description of what a successful HTTP response looks like. This is a hash of `headers` and `body` content:

- `headers`  
    Any standard HTTP header can be expected here, but most implementations won't
    need this information. `status` is provided as an example.
  - `status`: The numeric http code the response should match, eg `200`
- `body`
  - `contains`: A plain string that should be present in the body. This is
    preferred over `matches` unless a more complex rule is necessary.
  - `matches`: A regular expression bounded by plain string delimiters ("") for portability. It's preferable to provide a pattern (if one is needed) that can
    be matched case-insensitively and on one line.

---

## Types of steps

### visit

The value of a visit step is just a `string` url.

### iframe

The purpose of this is to access elements on a form that may be embedded in an iframe. Using the selector, you can switch the frame of the browser to the given iframe. Using the boolean "back" you can return to the parent frame instead of entering a new iframe

### find

The value of a find step is just a string CSS selector which should be found on the page (and should be visible) before proceeding to execute more steps.

### wait

The value is a number of seconds to wait before performing another action. Find should be used if it all possible instead of wait.

### fill_in

The value of a fill_in step can be a single field, or a list of hashes defining a batch of fields to fill in at once, but should be defined as a list either way. Each hash describes a form field by a few attributes, many of which are common to most steps:

- `name`: The `name` HTML attribute of the field to be filled out.
- `selector`: It's expected that a specific CSS selector will be provided in
    addition to the `name` field, because it's possible that more than one field
    with the same name (`email`, for example) may be present on the page.
- `value`: Either a string value to enter into the form, or a 'variable'
    placeholder, such as `$EMAIL`. These placeholders are listed and explained in
    [variables.yaml](../support/variables.yaml) in the support folder of this repo.
    The leading dollar sign is used to help disambiguate these special values from
    an all-caps string value that might be intended to go directly into the form
    field.
- `required` (ironically, optional): This field will be present if a field *must*
    be filled out with a value in order for the form to be valid.
- `max_length` (optional): This field will be present if a field has a maximum length.
    It's very useful where max length is only enforced server-side. Must be within an `options` block, like so:

    ```yaml
    - fill_in
      - name: test
        selector: "#id"
        value: $TEST
        required: true
        options:
          max_length: 1234
    ```
- `blacklist` (optional): This field is used to remove characters from text before it is entered. All removed characters will be replaced by a single whitespace. The value of this field can either be a series of characters or a regex. If using a regex, the value should start and end with a `/` character. Must be within an `options` block, like so:

    ```yaml
    - fill_in
      - name: test
        selector: "#id"
        value: $TEST
        required: true
        options:
          blacklist: "&'!."
    ```
- `wysiwyg` (optional): This field will be present if a field is a wysiwyg, such as ckeditor or tinymce. They cannot be filled in the same way as a typical textbox.

### remove
Removes DOM element from page.
- `selector`: Selector for the DOM element to remove

### picker
Picks element using key-selector mapping and a provided key value.

Example:
```yaml
- picker:
  - name: topic
    value: $TOPIC
    required: true
    options:
      Abortion: "#field_302E8A41-000D-419E-991E-40C7CB96F97C_1"
      Animals: "#field_302E8A41-000D-419E-991E-40C7CB96F97C_3"
```
For this to select `Animals` on the page, the `$TOPIC` value would be set to `Animals`.



### recaptcha

Clicks the reCAPTCHA checkbox to open the challenge, then fetchs either the image or audio challenge (depending on configuration).
- `grid_selector`: Selector for the element that contains the grid of 3x3 images
- `img_selector`: Selector the the 3x3 grid image
- `phrase_selector`: Selector for the element that contains the challenge phrase
- `phrase_selector_fallback`: Selector the element that contains the challenge phrase. Used to handle case when there is not a canonical image with the phrase, because that causes the selector to change.
- `simple_img_selector`: Selector for image, in the event a simple reCAPTCHA challenge is presented
- `simple_textbox_selector`: Select for textbox, in the event a simple reCAPTCHA challenge is presented
- `audio_selector`: Selector for the download link for the audio file
- `audio_response_selector`: Selector for the response to the audio challenge
- `audio_switch_selector`: Selector for the button to switch between image and audio challenges
- `checkbox_iframe_selector`: Selector for IFrame that contains the initial checkbox (to begin reCAPTCHA)
- `checkbox_selector`: Selector for checkbox to click to begin reCAPTCHA
- `main_iframe_selector`: Selector for IFrame that contains main challenge content
- `verify_selector`: Selector for the verify button, to submit reCAPTCHA
- `types`: Array value, containing `image` and/or `audio`, to indicate which challenge types the yaml supports

For **all** challenges, the following selectors are **required**:
- `checkbox_iframe_selector`
- `checkbox_selector`
- `main_iframe_selector`
- `verify_selector`
- `types`

For **image** challenges, the following selectors are also **required**:
- `grid_selector`
- `img_selector`
- `phrase_selector`
- `phrase_selector_fallback`
-  `simple_img_selector` -- needed to support simple reCAPTCHA challenges
-  `simple_textbox_selector` -- needed to support simple reCAPTCHA challenges

For **audio** challenges, the following selectors are also **required**:
- `audio_selector`
- `audio_response_selector`
- `audio_switch_selector`

**A note on CAPTCHAs (besides reCAPTCHA V2)**

Contact forms may present a captcha challenge, which of course is difficult to deal with in an automated fashion. CAPTCHAs should be handled as `fill_in` fields with the variable `$CAPTCHA_SOLUTION` as the value. These fields should also describe a `captcha_selector` key for retrieving the captcha image and returning it to a solver of the implementer's choosing.

### check/uncheck/choose

These steps can also either list one or many hashes. It should be expected that a single form can be filled out with many steps until a `click_on` step is encountered, at which time the form should be submitted.

The attributes of these steps are the same as those of `fill_in`, and should be treated as such with the exception of `value`. In a checkbox or radio button context, `value` describes the actual `value` attribute of the checkbox that should be checked/unchecked/chosen, in case several have the same `name` attribute.

### select

Like the other input-related steps, `select`s can list either one or many hashes. Attributes are the same as `fill_in` with the addition of `options`, a list of the possible options which can be selected. If the `value` attributes of the select's options are obscure abbreviations or otherwise non-human-readable, the value of `options` can be a hash where the key is the text that appears in the select box when the option is selected, and the value is the option's `value` attribute. In cases where the options are common across several members' forms, a *constant* may be used as a placeholder. Available constants are listed in [constants.yaml](../support/constants.yaml) in this repository. Currently the only available constants are a list of the postal codes of the 50 US states plus DC, and the full list of states and territories. The constants encountered in options lists comprise the keys in `constants.yaml` so the resulting constants hash can be indexed directly with them.

### click_on

A click_on step terminates the preceding list of input-related steps, by submitting the web form. It is a list containing a hash with only two possible keys: `selector` and `value`. `selector` is the CSS selector for finding the button or link to click, and `value` is the HTML value attribute if present, both to disambiguate and for the benefit of clients which may be POSTing directly instead of using a headless browser, though this is not recommended. `selector` is the only attribute you must provide/should expect to be guaranteed.

### javascript

A javascript step allows running arbitrary client-side javascript on the page that yamltron is on. The `selectors` property is used to inject an array of elements. The `values` property is used to inject an array of values, which is useful for injecting our $ variables, such as `$PHONE`, etc.

Example:
```yaml
- javascript:
  - name: topic
    selectors: [ "#request_custom_fields_27999287" ]
    commands: [ "elements[0].value = 'other_not_listed'"]
```

---

## Examples

To put it all together, let's do a google search for our last name as an example:

```YAML
bioguide: #(well, it's google, so there isn't one)
contact_form:
  method: GET
  action: http://google.com/search
  steps:
    - visit: http://google.com
    - fill_in:
      - name: q
        selector: "#gbqfq"
        value: $NAME_LAST
        required: true
    - click_on:
      - selector: "#gbqfba"
  success:
    body:
      contains: "results ("
```

Here is a list of examples that may help you:
- [Handling Captchas](https://github.com/unitedstates/contact-congress/blob/master/support/recaptcha-noscript.yaml)
- [Handling non human readable options](https://github.com/unitedstates/contact-congress/blob/master/members/S000033.yaml)
- [Handling radio buttons](https://github.com/unitedstates/contact-congress/blob/master/members/H001049.yaml) - checkboxes are the same concept except with "- check"
- [Finding created forms](https://github.com/unitedstates/contact-congress/blob/master/members/I000024.yaml)
