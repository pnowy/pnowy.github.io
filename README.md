### DEVELOPMENT

```
npm install
bower install
```

### NEW JS LIBS

Bower is not used for generate index.html so the libraries downloaded by bower
need to be copied manually to assets and linked on _includes/head.html

### BUILD WITH DRAFTS

`jekyll.bat build --drafts`

### BUILD FOR DEV

`jekyll.bat build --config=_config.yml,_config_dev.yml`
or
`devserve.bat`