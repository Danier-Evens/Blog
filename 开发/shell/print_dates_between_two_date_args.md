* shell脚本接受时间范围段参数，例如2017-05-01 ~ 2017-05-05，输出 2017-05-01、2017-05-02、2017-05-03、2017-05-04、2017-05-05所有天数。

    ```
    start=`date --date="$1"  +%s`
    end=`date --date="$2"  +%s`
    #当前时间
    current_format=`date --date='0 days ago' +%Y-%m-%d`
    echo "current_format=$current_format"

    #转化为秒 计算当前天跟数据的结束时间相差的天数
    current=`date --date="$current_format"  +%s`
    diff_current_end=$((($current-$end)/3600/24))
    echo "diff_current_end=$diff_current_end"

    #输入参数时间范围相差的天数
    diff_end_start=$((($end-$start)/3600/24))
    echo "diff_end_start=$diff_end_start"

    for ((i=0; i<=$diff_end_start; i++))
    do
      days=$(( $diff_end_start + $diff_current_end - i ))
      currentday=`date --date="$days days ago" +%Y-%m-%d`
      echo ${currentday}
    done
    ```

    ![如图](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/date_print.png)