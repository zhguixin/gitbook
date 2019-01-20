Button的样式

原生的样式：

```xml
    <style name="Widget.Material.Button">
        <item name="background">@drawable/btn_default_material</item>
        <item name="textAppearance">?attr/textAppearanceButton</item>
        <item name="minHeight">48dip</item>
        <item name="minWidth">88dip</item>
        <item name="stateListAnimator">@anim/button_state_list_anim_material</item>
        <item name="focusable">true</item>
        <item name="clickable">true</item>
        <item name="gravity">center_vertical|center_horizontal</item>
    </style>
```

公共控件的样式：

```xml
    <style name="Widget.ZTE.Light.Button" parent="@android:style/Widget.Material.Button">
		<item name="android:background">@drawable/btn_default_material</item>
		<!--item name="android:textSize">16sp</item-->
		<item name="android:textColor">@color/primary_text_zte_light</item>
		<!--item name="android:stateListAnimator">@null</item-->
    </style>
```

