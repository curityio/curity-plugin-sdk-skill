# Templating

## 1. Template path
View templates should be placed relative to the resources directory in `templates/<plugin-type>/<plugin-impl-type>`. So if you are building an authenticator with the name `my-good-authn`, the base path should be `templates/authenticator/my-good-authn`.

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

**Example:** For template `templates/authenticator/username-password/authenticate/get.vm`:
- `${_templatePrefix}` resolves to `authenticator.username-password.authenticate`
- `#message("${_templatePrefix}.username.label")` looks up `authenticator.username-password.authenticate.username.label`

**Message files** are stored in `src/main/resources/messages/<locale>/` directory:
- English: `messages/en/authenticator.properties`
- Swedish: `messages/sv/authenticator.properties`
- Messages can be split across multiple property files for organization

Example message file (`messages/en/authenticator.properties`):
```properties
authenticator.username-password.authenticate.username.label=Username
authenticator.username-password.authenticate.password.label=Password
authenticator.username-password.authenticate.submit.label=Login
```

Example `get.vm` with localization:

```velocity
#define($_body)
    #if($_error)
    <div class="alert alert-danger">$_error</div>
    #end
    
    <form method="post">
        <label for="username">#message("${_templatePrefix}.username.label")</label>
        <input type="text" id="username" name="username" value="$!_username" 
               placeholder="#message("${_templatePrefix}.username.placeholder")" />
        
        <label for="password">#message("${_templatePrefix}.password.label")</label>
        <input type="password" id="password" name="password" 
               placeholder="#message("${_templatePrefix}.password.placeholder")" />
        
        <button type="submit">#message("${_templatePrefix}.submit.label")</button>
    </form>
#end
#parse('layouts/default')
```

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
