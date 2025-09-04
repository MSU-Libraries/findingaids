Theme Setup 
--------------------------------------------------  
1. Clone the theme into `archivesspace/plugins/[plugin-name]` 

2. Open `archivesspace/config/config.rb` and add `[plugin-name]` to the `AppConfig[:plugins]` section.  
Example (when plugin is named `msul_theme`): `AppConfig[:plugins] = ['msul_theme', 'local', 'lcnaf' ]`.

3. Restart ArchivesSpace to apply changes to the theme


For more information on plugins see the [official documentation](https://archivesspace.github.io/tech-docs/customization/plugins.html).
