kotlin、AAC笔记
================

###ViewBinding
```groovy
android {
    viewBinding {
        enabled = true
    }
}

-------
demo_binding.xml
-------

var binding = DemoBindingBinding.inflate(layoutInflater)
setContentView(binding.root)

```

ViewBinding单纯用来减少findViewById的模板代码。

###ViewModel
