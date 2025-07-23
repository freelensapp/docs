# LensRendererExtension class

The `LensRendererExtension` class extends the base `LensExtension` class and serves as a central place to register various UI components and features for a FreeLens extension.
Each property in this class is an array that holds registrations for different types of UI elements or functionality. These registrations are then integrated into specific parts of the FreeLens UI. For example:

```javascript linenums="1"
export class LensRendererExtension extends LensExtension {
  globalPages: PageRegistration[] = [];
  clusterPages: PageRegistration[] = [];
  clusterPageMenus: ClusterPageMenuRegistration[] = [];
  clusterFrameComponents: ClusterFrameChildComponent[] = [];
  kubeObjectStatusTexts: KubeObjectStatusRegistration[] = [];
  appPreferences: AppPreferenceRegistration[] = [];
  appPreferenceTabs: AppPreferenceTabRegistration[] = [];
  entitySettings: EntitySettingRegistration[] = [];
  statusBarItems: StatusBarRegistration[] = [];
  kubeObjectDetailItems: KubeObjectDetailRegistration[] = [];
  kubeObjectMenuItems: KubeObjectMenuRegistration[] = [];
  kubeWorkloadsOverviewItems: WorkloadsOverviewDetailRegistration[] = [];
  commands: CommandRegistration[] = [];
  welcomeMenus: WelcomeMenuRegistration[] = [];
  catalogEntityDetailItems: CatalogEntityDetailRegistration<CatalogEntity>[] = [];
  topBarItems: TopBarRegistration[] = [];
  additionalCategoryColumns: AdditionalCategoryColumnRegistration[] = [];
  customCategoryViews: CustomCategoryViewRegistration[] = [];
  kubeObjectHandlers: KubeObjectHandlerRegistration[] = [];

  ...
}
```

Going further in the doc, you can find an example for each of these.