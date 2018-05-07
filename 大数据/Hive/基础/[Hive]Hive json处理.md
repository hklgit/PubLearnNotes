```
select sum(get_json_object(line,'$.orderSize'))
from tmp_adv_show_click_order
where get_json_object(line,'$.isClicked') = 'true'
and get_json_object(line,'$.channelId') like '%toutiao%'
and dt = '20180503';
```
