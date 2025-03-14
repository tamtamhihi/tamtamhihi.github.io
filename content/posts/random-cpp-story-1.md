---
title: 'Random CPP story #1'
date: 2025-03-14
tags: ["cpp"]
---

Hai câu chuyện thú dị nho nhỏ mà mình vừa gặp liên quan đến mutex/lock và CPP nên ghi lại để mắc công quên.

## Deadlock
Chuyện là trong công việc thì ngoài làm project chính, thỉnh thoảng mọi người cũng hay commit những cái thay đổi nho nhỏ để clean up hoặc improve codebase. Mới vài ngày trước thì đồng nghiệp mình có push một cái commit clean up như sau.

Trạng thái ban đầu:
```cpp
void func() {
  if (a_condition_that_is_always_true) {
    <SOME_CODE>
  }
  doSomething();
}
```
Sau khi thay đổi:
```cpp
void func() {
  <SOME_CODE>
  doSomething();
}
```

Một thay đổi nhìn khá hiển nhiên và vô hại. Thực ra thì mình cũng hay có mấy commit kiểu vầy, tại mỗi lần dùng flag để roll out cái gì mới, roll out xong (nên mới có cái `a_condition_that_is_always_true`) là chẳng ai thèm nhớ (hay quan tâm) để dọn.

Commit này cũng được merge, nhưng trước khi nó được đẩy lên production thì có một đồng nghiệp khác nhắn vào group chat là "tao nghĩ mình nên revert cái commit này vì nó có thể gây ra deadlock".

Mình cũng tò mò vào xem thử, thì hoá ra là đoạn `<SOME_CODE>` đang access vào một object được gắn với mutex (vì object này có thể được access từ nhiều thread, see more [here](https://github.com/facebook/folly/blob/main/folly/docs/Synchronized.md)), cụ thể là đang lấy write lock trên object này để chỉnh sửa. Và bên trong hàm `doSomething` cũng có đoạn code access vào object này nhưng là lấy read lock. Ban đầu, khi còn cái if condition, thì write lock được obtained bên trong cái scope của if nên khi thoát ra khỏi đoạn này thì write lock đã bị destroyed, và hàm `doSomething()` có thể tiếp tục lấy read lock mà không có vấn đề gì cả. Nhưng vì clean up và xoá đi cái scope của if nên write lock vẫn còn đó và khi cố gắng lấy read lock thì sẽ bị dính deadlock 😵‍💫 

Khổ cái là chắc không ai để ý bên trong `doSomething` lại lấy lock, và cũng không để ý vụ write lock chưa được release. Với cả nói đúng hơn là cái change này không được test (chắc do thấy vô hại quá). 

Và commit này sau đó cũng được revert. Thật ra vẫn có giải pháp để clean đi mớ condition kia, đó là wrap cái write lock bên trong một cái unnamed scope:
```cpp
void func() {
  {
    <SOME_CODE>
  }
  doSomething();
}
```


## Garbage collector

Cái này thì mình tình cờ đọc được đoạn code này thấy khá hay:

```cpp
void filter_some_keys_out_of_map() {
  vector<Entry> entriesCollector;
  locked_map = <get write lock on map>;
  for (key : locked_map) {
    if (key should be filtered out) {
      Entry entry; // empty entry
      swap(entry, locked_map[key]);
      entriesCollector.emplace_back(std::move(entry));
      locked_map.erase(key);
    }
  }
}
```

Đại loại là hàm này muốn clean một số keys bên trong cái map (cũng được gắn với mutex). Nếu suy nghĩ đơn giản thì mình có thể làm như sau:
```cpp
void filter_some_keys_out_of_map() {
  locked_map = <get write lock on map>;
  for (key : locked_map) {
    if (key should be filtered out) {
      locked_map.erase(key);
    }
  }
}
```
Vấn đề nằm ở chỗ là khi mình gọi `erase(key)` trên map thì sẽ trigger destructor của `Entry` tại key đó. Nghĩa là ở phiên bản bên dưới thì destructor của tất cả các `Entry` mình erase đều sẽ được gọi **while** write lock vẫn còn đó (trong trường hợp này thì write lock chỉ bị destroyed khi kết thúc hàm). Và vì Entry là những object tạm coi là khá lớn, nên việc giữ write lock trong quá trình dọn dẹp này có vẻ không được tối ưu.

Vì vậy nên mới có phiên bản cồng kềnh hơn ở trên, đó là trước khi gọi `erase(key)` thì mình sẽ swap vào đây một cái empty `Entry`. Dù destructor của nó cũng vẫn sẽ bị trigger thôi nhưng vì nó nhỏ nên quá trình này sẽ diễn ra nhanh hơn. Còn cái `Entry` bự đã được đưa vào một cái vector bên ngoài `entriesCollector`. Khi kết thúc hàm này, vì write lock được declare sau nên sẽ bị destroyed trước, và sau đó mới tới `entriesCollector` (và các object bên trong) => việc gọi destructor trên các `Entry` objects lớn đã được move ra khỏi thời gian giữ write lock.
