# Templating

## 1. Template path
View templates should be placed relative to the resources directory in `templates/<plugin-type>/<plugin-impl-type>/<request-handler-path>/`. 

**Examples:**
- Authenticator: `templates/authenticator/username-password/authenticate/get.vm`
- Authentication Action: `templates/authentication-action/info-message/index/get.vm`

The full path includes:
1. Plugin type (e.g., `authenticator`, `authentication-action`)
2. Plugin implementation type (e.g., `username-password`, `info-message`)
3. Request handler path (e.g., `authenticate`, `index`)
4. HTTP method file (e.g., `get.vm`, `post.vm`)

**Template files must use the `.vm` extension** as they use Apache Velocity template syntax.

## 2. Template contents
The templates should use `#define($_body)` to define the main content and `#parse('layouts/default')` to include the standard layout.

**Templates should focus only on plugin-specific components:**
- Forms and input fields
- Error messages and validation feedback
- Information display relevant to the plugin functionality
- Do not include page headlines, navigation, or general layout elements (these are provided by `default/layout`)

## 3. Velocity syntax
- Use `$variable` or `$!variable` to output variables (the `!` prevents errors for null values)
- Use `#if($condition)` ... `#end` for conditionals
- Use `#foreach($item in $collection)` ... `#end` for loops
- Use `#message("message.key")` to output localized messages

## 4. Localization with message keys
Templates use the `#message()` directive to support localization. The `$_templatePrefix` variable is automatically provided by the server based on the template path.

**Message key pattern:**
```
#message("${_templatePrefix}.key.for.message")
```

**How `${_templatePrefix}` works:**

For template `templates/authentication-action/info-message/index/get.vm`:
- `${_templatePrefix}` resolves to `authentication-action.info-message.index`
- `#message("${_templatePrefix}.content")` looks up `authentication-action.info-message.index.content`

**Message files folder structure:**

Message files are stored in `src/main/resources/messages/<locale>/<plugin-type>/<plugin-impl-type>/` directory, mirroring the template structure.

**Structure:**
```
messages/
  en/                                    # Locale (en, sv, etc.)
    authentication-action/               # Plugin type
      info-message/                      # Plugin implementation type
        messages.properties              # Message file (can have any name ending in .properties)
```

**Message key structure:**

Since the folder path already includes `<locale>/<plugin-type>/<plugin-impl-type>`, message keys only need the **request handler path and property name**.

Example message file (`messages/en/authentication-action/info-message/messages.properties`):
```properties
# Keys start with request handler path, not full plugin path
index.content=This is an important informational message.
index.button.continue=Continue
```

**Why keys are short:**
- Template path: `templates/authentication-action/info-message/index/get.vm`
- Message folder: `messages/en/authentication-action/info-message/`
- `${_templatePrefix}`: `authentication-action.info-message.index`
- The server combines folder path + file keys to resolve full message key

**Complete example:**

Template at `templates/authentication-action/info-message/index/get.vm`:
```velocity
#define($_body)
    <div class="alert alert-info mb2">
        <p>#message("${_templatePrefix}.content")</p>
    </div>
    
    <form method="post" action="">
        <button type="submit" class="button button-fullwidth mt2">
            #message("${_templatePrefix}.button.continue")
        </button>
    </form>
#end
#parse('layouts/default')
```

Messages at `messages/en/authentication-action/info-message/messages.properties`:
```properties
# Request handler path is 'index', so keys start with 'index.'
index.content=This is an important informational message.
index.button.continue=Continue
```

How it resolves:
- `${_templatePrefix}` = `authentication-action.info-message.index`
- `#message("${_templatePrefix}.content")` looks up `authentication-action.info-message.index.content`
- Server finds file at `messages/en/authentication-action/info-message/*.properties`
- Server looks for key `index.content` in that file

## 5. Naming Convention for Model Variables

**Convention:** Variables provided by the server or plugin should use a **leading underscore** (`_`) prefix. This distinguishes server-provided variables from variables created within templates.

**Server-provided variables (use underscore):**
- `$_error` - Error messages from the server
- `$_username` - Username from failed authentication
- `$_templatePrefix` - Template path prefix (automatically provided)
- Any data passed via `ResponseModel.templateResponseModel(viewData, ...)`

**Template-local variables (no underscore):**
```velocity
#set($counter = 0)
#set($localVar = "value")
```

**Example from request handler:**
```kotlin
// In request handler - use underscore prefix for model data
val viewData = mapOf(
    "_error" to "Invalid username or password",
    "_username" to username
)
response.setResponseModel(
    ResponseModel.templateResponseModel(viewData, "authenticate/get"),
    Response.ResponseModelScope.ANY
)
```

**Example in template:**
```velocity
#if($_error)
    <div class="error">$_error</div>
#end
<input type="text" name="username" value="$!_username" />
```

## 6. Styling with Curity CSS Classes

Curity provides CSS classes for consistent styling of forms and UI elements. Use these classes to ensure your plugin matches the server's look and feel.

**Common CSS Classes:**

**Form containers:**
- `form-field` - Wrapper for input field with label and icon

**Input fields:**
- `block` - Display as block element
- `full-width` - 100% width
- `field-light` - Light-themed input field
- `mb1`, `mb2`, `mb3` - Margin bottom (1, 2, 3 units)
- `mt0`, `mt1`, `mt2`, `mt3` - Margin top (0, 1, 2, 3 units)

**Buttons:**
- `button` - Base button class
- `button-fullwidth` - Full-width button

**Typography:**
- `center` - Center-align text

**Icons:**
- `form-field-icon` - Icon inside form field
- `icon` - Base icon class
- `ion-ios-person` - Person/user icon (Ionicons)
- `ion-ios-locked` - Lock/password icon (Ionicons)
- `ion-android-person-add` - Add person icon

**Layout:**
- `clearfix` - Clear floats
- `py2` - Padding top and bottom (2 units)

**Alerts:**
- `alert alert-danger` - Alert with an error level
- `alert alert-warning` - Alert with a warning level
- `alert alert-info` - Alert with a info level

**Example form with Curity CSS:**

```velocity
#define($_body)
    #if($_error)
    <div class="alert alert-danger">
        <span>$_error</span>
    </div>
    #end
    
    <form method="post" action="">
        <h1 class="mt0 center">#message("${_templatePrefix}.heading")</h1>
        
        <div class="form-field">
            <label for="username">#message("${_templatePrefix}.username.label")</label>
            <input 
                type="text" 
                id="username" 
                name="username" 
                value="$!_username"
                autocorrect="off" 
                spellcheck="false" 
                class="block full-width mb1 field-light" 
                autocapitalize="none"
                autocomplete="username"
                placeholder="#message("${_templatePrefix}.username.placeholder")"
                autofocus
            />
            <i class="form-field-icon icon ion-ios-person"></i>
        </div>

        <div class="form-field">
            <label for="password">#message("${_templatePrefix}.password.label")</label>
            <input 
                type="password" 
                id="password" 
                name="password" 
                class="block full-width mb1 field-light"
                autocomplete="current-password"
                placeholder="#message("${_templatePrefix}.password.placeholder")"
            />
            <i class="form-field-icon icon ion-ios-locked"></i>
        </div>

        <button type="submit" class="button button-fullwidth mt2">#message("${_templatePrefix}.submit.label")</button>
    </form>
#end
#parse('layouts/default')
```

**Best practices:**
- Always use `form-field` wrapper for inputs with labels and icons
- Add appropriate icons using `form-field-icon` and Ionicons classes
- Use `block full-width` for input fields to ensure consistent sizing
- Apply `button button-fullwidth` for primary action buttons
- Use spacing classes (`mt`, `mb`, `py`) for consistent margins and padding
- Use `alert alert-info` classes to show information to the user
