# App preferences

## What is appPreferences?
App preferences allow you to add new menu entries to the global settings page.
You can access this page by clicking the sandwich menu in the top-left corner and selecting `Preferences`.
Alternatively, you can press `Ctrl + ,`

<figure markdown="span">
  ![openClusterSettings](images/app-preferences/open-app-preferences.png)
</figure>

You will see the new component here
<figure markdown="span">
  ![openClusterSettings](images/app-preferences/app-preferences.png)
</figure>

# How to use it
## Interface

This is the appPreferences interface
```javascript
appPreferences: AppPreferenceRegistration[] = [];
```

AppPreferenceRegistration is defined as follows
```javascript linenums="1"
interface AppPreferenceRegistration {
  title: string;
  id?: string;
  showInPreferencesTab?: string;
  components: AppPreferenceComponents;
}
```

## Hands on
To add a new setting menu entry in cluster settings just add this code of block to your `renderer/index.tsx` file

```javascript linenums="1"
appPreferences = [
  {
    title: "My Settings",
    components: {
      Input: () => <div>HELLO!</div>,
      Hint: () => <span>Span message</span>,
    },
  },
];
```

!!! warning "After compiling and installing the plugin, remember to close Freelens from the tray icon and reopen it, or you won’t see the new menu entries"
