```ruby
    relations = [
      [1, 2],
      [2, 3],
      [4, 5],
      [8, 9],
      [10, 1]
    ]

    relations = relations + relations.map{|ar| [ar.last, ar.first] }

    inc = relations.count * 2
    group_id = 1
    gc = {}
    relations.each do |r|
      r.each do |e|
        if gc[e].nil?
          gc[e] = group_id
        else
          pre_group_id = gc[e]
          gc.select{|k, v| v == pre_group_id }.keys.each do |key|
            gc[key] = group_id
          end
        end
      end
      group_id += inc
    end

    class Hash
      def safe_invert
        self.each_with_object({}){|(k,v),o|(o[v]||=[])<<k}
      end
    end

    puts gc.safe_invert
```
