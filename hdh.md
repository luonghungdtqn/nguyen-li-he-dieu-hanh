# Phân tích Thuật toán Điều độ CPU: Round-Robin (RR) và SRTN

Tài liệu này trình bày chi tiết về hai thuật toán điều độ tiến trình phổ biến trong môn Nguyên lý Hệ điều hành: Quay vòng (Round-Robin) và Thời gian còn lại ngắn nhất (SRTN).

---

## 1. Thuật toán Quay vòng (Round-Robin - RR)

### Ý tưởng
Thuật toán RR được thiết kế đặc biệt cho các hệ thống chia sẻ thời gian (time-sharing systems). 
* Hệ thống định nghĩa một khoảng thời gian nhỏ gọi là lượng tử thời gian (time quantum, ký hiệu là $q$), thường từ 10 - 100 milliseconds.
* Hàng đợi Sẵn sàng (Ready Queue) được tổ chức theo cơ chế FIFO (First-In, First-Out).
* Bộ điều phối CPU sẽ chọn tiến trình đầu tiên trong hàng đợi Sẵn sàng, cấp phát CPU cho nó thực thi trong khoảng thời gian tối đa bằng $q$.
* **Nếu tiến trình kết thúc trước $q$:** Nó tự động giải phóng CPU, hệ thống chọn tiến trình tiếp theo.
* **Nếu tiến trình chưa kết thúc sau $q$:** Hệ thống phát sinh ngắt (timer interrupt), tiến trình bị tiếm quyền (preempted) và bị đẩy xuống cuối hàng đợi Sẵn sàng. CPU được chuyển cho tiến trình tiếp theo.

### Lược đồ thuật toán (Dòng chảy Logic)
1. **Khởi tạo:** Nạp các tiến trình mới đến vào cuối hàng đợi `Ready_Queue`.
2. **Kiểm tra hàng đợi:** Nếu `Ready_Queue` rỗng và không còn tiến trình nào, kết thúc. Nếu có, lấy tiến trình $P_i$ ở đầu hàng đợi.
3. **Thực thi:** Cấp CPU cho $P_i$ một khoảng thời gian $t = \min(q, \text{Thời gian cần còn lại của } P_i)$.
4. **Cập nhật:** Tăng thời gian hệ thống thêm $t$. Giảm thời gian cần thực thi của $P_i$ đi $t$.
5. **Kiểm tra tiến trình mới:** Nạp bất kỳ tiến trình nào vừa mới đến (trong khoảng thời gian $t$ vừa chạy) vào `Ready_Queue`.
6. **Xử lý $P_i$:** 
   * Nếu $P_i$ đã chạy xong (Thời gian còn lại = 0), ghi nhận thời gian hoàn thành.
   * Nếu $P_i$ chưa xong, đẩy $P_i$ trở lại cuối `Ready_Queue`.
7. Quay lại **Bước 2**.

### Ví dụ minh họa
Giả sử có 3 tiến trình với thời gian đến (Arrival Time - AT) bằng 0 và lượng tử $q = 4$:

| Tiến trình | Thời gian thực thi (Burst Time - BT) |
|---|---|
| $P_1$ | 24 |
| $P_2$ | 3 |
| $P_3$ | 3 |

**Biểu đồ Gantt (Tiến trình thực thi):**
* **0 - 4:** $P_1$ chạy (còn 20).
* **4 - 7:** $P_2$ chạy xong và thoát (vì $BT = 3 < q$).
* **7 - 10:** $P_3$ chạy xong và thoát.
* **10 - 14:** $P_1$ tiếp tục chạy (còn 16)... Cứ thế cho đến khi $P_1$ xong tại thời điểm 30.

### Demo chương trình C++ (Round-Robin)

```cpp
#include <iostream>
#include <vector>
#include <queue>

using namespace std;

struct Process {
    int id;
    int at; // Arrival Time
    int bt; // Burst Time
    int rt; // Remaining Time
    int ct; // Completion Time
    int tat; // Turnaround Time
    int wt; // Waiting Time
};

void findRoundRobin(vector<Process>& proc, int n, int quantum) {
    queue<int> q;
    int current_time = 0;
    int completed = 0;
    vector<bool> in_queue(n, false);

    // Bắt đầu với tiến trình đầu tiên
    q.push(0);
    in_queue[0] = true;
    
    while (completed != n) {
        int idx = q.front();
        q.pop();
        
        int time_ran = min(quantum, proc[idx].rt);
        proc[idx].rt -= time_ran;
        current_time += time_ran;
        
        // Kiểm tra các tiến trình mới đến
        for (int i = 0; i < n; i++) {
            if (proc[i].at <= current_time && proc[i].rt > 0 && !in_queue[i] && i != idx) {
                q.push(i);
                in_queue[i] = true;
            }
        }
        
        if (proc[idx].rt > 0) {
            q.push(idx); // Đưa lại vào hàng đợi nếu chưa xong
        } else {
            proc[idx].ct = current_time;
            proc[idx].tat = proc[idx].ct - proc[idx].at;
            proc[idx].wt = proc[idx].tat - proc[idx].bt;
            completed++;
        }
    }
}

int main() {
    int n = 3, quantum = 4;
    vector<Process> proc = {
        {1, 0, 24, 24, 0, 0, 0},
        {2, 0, 3, 3, 0, 0, 0},
        {3, 0, 3, 3, 0, 0, 0}
    };

    findRoundRobin(proc, n, quantum);

    cout << "Round Robin Scheduling (q = " << quantum << ")\n";
    cout << "ID\tAT\tBT\tCT\tTAT\tWT\n";
    for (int i = 0; i < n; i++) {
        cout << proc[i].id << "\t" << proc[i].at << "\t" << proc[i].bt << "\t" 
             << proc[i].ct << "\t" << proc[i].tat << "\t" << proc[i].wt << "\n";
    }
    return 0;
}
```

---

## 2. Thuật toán Ưu tiên thời gian còn lại ngắn nhất (SRTN)

### Ý tưởng
SRTN (Shortest Remaining Time Next) là phiên bản có tiếm quyền (preemptive) của thuật toán Shortest Job First (SJF). 
* Tiêu chí chọn tiến trình: Tiến trình nào có **thời gian thực thi còn lại ngắn nhất** sẽ được cấp CPU.
* Tính tiếm quyền: Bất cứ khi nào có một tiến trình mới được đưa vào hàng đợi Sẵn sàng, hệ thống sẽ so sánh thời gian thực thi còn lại của tiến trình đang chạy với thời gian thực thi của tiến trình mới.
* Nếu tiến trình mới có thời gian yêu cầu ít hơn thời gian còn lại của tiến trình đang chạy, CPU sẽ bị thu hồi và cấp cho tiến trình mới.

### Lược đồ thuật toán (Dòng chảy Logic)
1. **Khởi tạo:** Đặt thời gian hệ thống $t = 0$.
2. **Quét tiến trình:** Tại thời điểm $t$, quét toàn bộ các tiến trình đã đến (Arrival Time $\le t$) và chưa hoàn thành.
3. **Tìm Min:** Tìm tiến trình $P_{min}$ có thời gian còn lại (Remaining Time - RT) nhỏ nhất.
4. **Thực thi:** Cấp CPU cho $P_{min}$ chạy trong 1 đơn vị thời gian (tăng $t$ lên 1, giảm RT của $P_{min}$ đi 1).
5. **Kiểm tra hoàn thành:** Nếu RT của $P_{min}$ bằng 0, ghi nhận thời gian hoàn thành (Completion Time = $t$). Tăng số tiến trình đã xong.
6. **Lặp lại:** Nếu số tiến trình hoàn thành chưa bằng tổng số tiến trình, quay lại **Bước 2**.

### Ví dụ minh họa
| Tiến trình | Thời gian đến (AT) | Thời gian thực thi (BT) |
|---|---|---|
| $P_1$ | 0 | 8 |
| $P_2$ | 1 | 4 |
| $P_3$ | 2 | 9 |
| $P_4$ | 3 | 5 |

**Biểu diễn quá trình (Biểu đồ Gantt):**
* **t=0:** $P_1$ đến, thực thi (Thời gian còn lại: $P_1=8$).
* **t=1:** $P_2$ đến ($BT=4$). Lúc này $P_1$ còn 7. Vì $4 < 7 \rightarrow$ **Tiếm quyền!** $P_2$ chiếm CPU.
* **t=2:** $P_3$ đến ($BT=9$). $P_2$ còn 3. $3 < 7 < 9 \rightarrow$ $P_2$ tiếp tục chạy.
* **t=3:** $P_4$ đến ($BT=5$). $P_2$ còn 2. $2 < 5 < 7 < 9 \rightarrow$ $P_2$ tiếp tục chạy.
* **t=5:** $P_2$ hoàn thành. Tiến trình có thời gian còn lại ngắn nhất hiện tại là $P_4$ (5 đơn vị). $P_4$ chạy.
* **t=10:** $P_4$ hoàn thành. Tiếp tục chọn $P_1$ (còn 7).
* **t=17:** $P_1$ hoàn thành. Chọn $P_3$ (còn 9).
* **t=26:** $P_3$ hoàn thành.

### Demo chương trình C++ (SRTN)

```cpp
#include <iostream>
#include <vector>
#include <climits>

using namespace std;

struct Process {
    int id;
    int at; // Arrival Time
    int bt; // Burst Time
    int rt; // Remaining Time
    int ct; // Completion Time
    int tat; // Turnaround Time
    int wt; // Waiting Time
};

void findSRTN(vector<Process>& proc, int n) {
    int completed = 0, current_time = 0;
    int min_rt = INT_MAX;
    int shortest = -1;
    bool check = false;

    // Lặp cho đến khi tất cả các tiến trình hoàn thành
    while (completed != n) {
        // Tìm tiến trình có RT nhỏ nhất trong các tiến trình đã đến
        for (int i = 0; i < n; i++) {
            if ((proc[i].at <= current_time) && (proc[i].rt < min_rt) && proc[i].rt > 0) {
                min_rt = proc[i].rt;
                shortest = i;
                check = true;
            }
        }

        if (!check) {
            current_time++;
            continue;
        }

        // Thực thi tiến trình ngắn nhất trong 1 đơn vị thời gian
        proc[shortest].rt--;
        min_rt = proc[shortest].rt;

        if (min_rt == 0) {
            min_rt = INT_MAX;
        }

        // Nếu tiến trình hoàn thành
        if (proc[shortest].rt == 0) {
            completed++;
            check = false;
            proc[shortest].ct = current_time + 1;
            proc[shortest].tat = proc[shortest].ct - proc[shortest].at;
            proc[shortest].wt = proc[shortest].tat - proc[shortest].bt;
        }
        current_time++;
    }
}

int main() {
    int n = 4;
    vector<Process> proc = {
        {1, 0, 8, 8, 0, 0, 0},
        {2, 1, 4, 4, 0, 0, 0},
        {3, 2, 9, 9, 0, 0, 0},
        {4, 3, 5, 5, 0, 0, 0}
    };

    findSRTN(proc, n);

    cout << "SRTN Scheduling\n";
    cout << "ID\tAT\tBT\tCT\tTAT\tWT\n";
    for (int i = 0; i < n; i++) {
        cout << proc[i].id << "\t" << proc[i].at << "\t" << proc[i].bt << "\t" 
             << proc[i].ct << "\t" << proc[i].tat << "\t" << proc[i].wt << "\n";
    }
    return 0;
}
```