# Câu 1: Thiết kế API

## 1. API lấy watchlists của tôi

```http
GET /api/v1/watchlists
```

**Response Payload**

```json
{
  "version": 3,
  "items": [
    {
      "symbol": "VCB",
      "note": "Theo dõi VCB"
    },
    {
      "symbol": "FPT",
      "note": "FPT có tiềm năng"
    }
  ]
}
```

## 2. API đồng bộ các operation từ client

```http
POST /api/v1/watchlists/sync
```

**Request Payload**

```json
{
  "baseVersion": 10,
  "operations": [
    {
      "operationId": "op-001",
      "type": "ADD",
      "symbol": "MWG"
    },
    {
      "operationId": "op-002",
      "type": "ADD",
      "symbol": "SSI"
    },
    {
      "operationId": "op-003",
      "type": "MOVE",
      "symbol": "SSI",
      "afterSymbol": "FPT"
    },
    {
      "operationId": "op-004",
      "type": "UPDATE_NOTE",
      "symbol": "MWG",
      "note": "Theo dõi"
    }
  ]
}
```

**Response Payload**

### Case 1: Nếu version server chưa bị thay đổi

```json
{
  "status": "APPLIED",
  "previousVersion": 10,
  "currentVersion": 11,
  "appliedOperationIds": [
    "op-001",
    "op-002",
    "op-003",
    "op-004"
  ],
  "watchlist": {
    "version": 11,
    "items": [
      {
        "symbol": "VNM",
        "note": ""
      },
      {
        "symbol": "FPT",
        "note": ""
      },
      {
        "symbol": "SSI",
        "note": ""
      },
      {
        "symbol": "MWG",
        "note": "Theo dõi"
      }
    ]
  },
  "conflicts": []
}
```

### Case 2: Nếu version server đã được cập nhật

```json
{
  "status": "STALE_VERSION",
  "baseVersion": 10,
  "serverVersion": 13
}
```

## 3. API lấy các thay đổi kể từ một version

```http
GET /api/v1/watchlists/changes?sinceVersion=10
```

**Response Payload**

```json
{
  "fromVersion": 10,
  "toVersion": 13,
  "operations": [
    {
      "operationId": "server-op-11",
      "version": 11,
      "type": "REMOVE",
      "symbol": "VNM"
    },
    {
      "operationId": "server-op-12",
      "version": 12,
      "type": "MOVE",
      "symbol": "VCB",
      "afterSymbol": null
    },
    {
      "operationId": "server-op-13",
      "version": 13,
      "type": "UPDATE_NOTE",
      "symbol": "FPT",
      "note": "Theo dõi thêm 3 ngày"
    }
  ]
}
```

## 4. API merge

```http
POST /api/v1/watchlists/merge
```

**Request Payload**

```json
{
  "baseVersion": 10,
  "expectedServerVersion": 13,
  "operations": [
    {
      "operationId": "op-001",
      "type": "ADD",
      "symbol": "MWG"
    },
    {
      "operationId": "op-002",
      "type": "ADD",
      "symbol": "SSI"
    }
  ]
}
```

**Response Payload**

### Case 1: Merge k bị conflict

```json
{
  "status": "MERGED",
  "baseVersion": 10,
  "previousServerVersion": 13,
  "currentVersion": 14,
  "watchlist": {
    "version": 14,
    "items": [
      {
        "symbol": "VCB",
        "note": ""
      },
      {
        "symbol": "FPT",
        "note": "Chờ công bố kết quả kinh doanh"
      },
      {
        "symbol": "MWG",
        "note": ""
      },
      {
        "symbol": "SSI",
        "note": ""
      }
    ]
  },
  "conflicts": []
}
```

### Case 2: Merge bị conflict

```json
{
  "status": "CONFLICT_REQUIRES_RESOLUTION",
  "baseVersion": 10,
  "serverVersion": 13,
  "conflicts": [
    {
      "conflictId": "conflict-001",
      "symbol": "VNM",
      "type": "REMOVE_VS_UPDATE_NOTE",
      "serverChange": {
        "operation": "REMOVE",
        "operationId": "op-server-001"
      },
      "clientChange": {
        "operation": "UPDATE_NOTE",
        "operationId": "op-client-002",
        "note": "chờ về 80 thì múc"
      },
      "allowedResolutions": [
        "KEEP_EDITED_ITEM",
        "KEEP_DELETED"
      ]
    }
  ]
}
```

## 5. API giải quyết conflict

```http
POST /api/v1/watchlists/conflicts/{conflictId}/resolve
```

**Request Payload**

```json
{
  "expectedServerVersion": 13,
  "resolution": "KEEP_EDITED_ITEM"
}
```

**Response Payload**

```json
{
  "status": "RESOLVED",
  "currentVersion": 14,
  "resolvedConflictId": "conflict-001"
}
```

# Câu 2:

Link GitHub: [SCEX_TEST](https://github.com/Tuan2110/SCEX_TEST)

# Câu 3:

## Tình huống A:

Tôi lựa chọn giữ lại mã sau khi merge vì với đặc thù Watchlist của nền tảng trading, tôi ưu tiên bảo toàn ghi chú mới, bởi vì ghi chú có thể chứa thông tin cá nhân quan trọng về chiến lược giao dịch. Nếu sau đó không cần thì có thể tiếp tục xoá đi, còn chọn xoá mã sau khi merge thì sẽ dẫn đến nguy cơ bị bỏ lỡ mất thông tin quan trọng và ảnh hưởng lớn đến user

## Tình huống B:

So sánh trạng thái cuối cho biết hai thiết bị khác nhau ở đâu, nhưng không cho biết vì sao chúng khác nhau. Đây là giới hạn về thông tin đầu vào dẫn đến một số hệ quả sau:

- Có thể khôi phục dữ liệu đã bị xóa ngoài ý muốn.
- Có thể xóa mất dữ liệu vừa được thêm.

> Không có thuật toán nào đảm bảo merge đúng ý định người dùng nếu chỉ nhận hai danh sách cuối cùng.

# Câu 4:

## Log mobile

```text
10:02:11 network=OFFLINE
10:03:40 local op: ADD "HPG" (queued)
10:04:02 local op: ADD "SSI" (queued)
10:05:17 local op: ADD "VCI" (queued)
10:41:00 network=ONLINE
10:41:02 SYNC → POST /watchlists/sync
baseVersion=17, operations=3
10:41:02 SYNC ← STALE_VERSION
client=17, server=20
10:41:03 MERGE → POST /watchlists/merge
base=17, expectedServer=20
10:41:03 MERGE ← MERGED newVersion=21
conflicts=0, operations=3
10:41:03 local queue: ACKED (3 ops)
```

## Log server

```text
10:20:45
web REMOVE "VNM" → v18
10:22:03
web REMOVE "FPT" → v19
10:23:11
web MOVE "MWG" to HEAD → v20
10:41:02
mobile SYNC base=17, server=20
VERSION_MISMATCH detected
Rejected direct write
10:41:03
MERGE started: base=17, remote=20
base: [VNM,FPT,MWG]
server: [MWG]
local: [VNM,FPT,MWG,HPG,SSI,VCI]
Non-conflicting changes detected
Conflicts=0
Merged: [MWG,HPG,SSI,VCI]
OCC CAS version 20 → 21: SUCCESS
Commit completed → v21
```

## Giải thích:

Ban đầu version 17 và list là VNM FPT MWG
Base version 17:

```text
VNM FPT MWG
```

Sau đó mobile thêm 3 mã vào local và version vẫn là 17
Mobile — local:

```text
VNM FPT MWG HPG SSI VCI
```

Server làm 3 thao tác xoá VNM -> version 18 , xoá FPT -> version 19, Kéo MWG về vị trí đầu tiên -> version 20
Server version 20:

```text
MWG
```

Áp dụng các solution đã làm ở các câu trên Merge 3 thao tác thêm vào danh sách .Do không có conflict nên không log gì thêm
Sau khi merge:

```text
MWG HPG SSI VCI
```

# Câu 5:

Dạ chị Đào, em hiểu ý chị. Nếu mình chỉ lấy danh sách được gửi lên sau cùng thì đúng là sẽ làm nhanh hơn và tiết kiệm được thời gian ban đầu.

Nhưng có một điểm em muốn chị cân nhắc: Danh sách được gửi lên sau cùng chưa chắc đã chứa những quyết định mới nhất của khách hàng.
Ví dụ, buổi sáng khách mở điện thoại khi không có mạng và thêm ba mã chứng khoán. Buổi chiều, khách mở máy tính, xóa hai mã không còn quan tâm và sắp xếp lại danh sách.

Đến tối, điện thoại có mạng trở lại và gửi những thay đổi từ buổi sáng lên. Nếu mình chỉ lấy danh sách gửi đến sau cùng, hai mã khách đã xóa sẽ xuất hiện trở lại, còn cách sắp xếp trên máy tính cũng bị mất.

Với khách hàng, điều đó có nghĩa là họ đã thực hiện đúng mọi thao tác nhưng hệ thống lại tự ý thay đổi quyết định của họ.

Đặc biệt với khách hàng sử dụng nền tảng giao dịch, danh sách theo dõi và ghi chú có thể gắn với những quyết định đầu tư quan trọng. Việc mất thông tin không chỉ gây khó chịu mà còn làm khách hàng mất niềm tin vào sản phẩm.

Em nghĩ mục tiêu của mình không phải là chọn thiết bị nào gửi dữ liệu sau cùng, mà là giữ được những thay đổi khách hàng thực sự muốn thực hiện.

Em nghĩ mục tiêu của mình không phải là chọn thiết bị nào gửi dữ liệu sau cùng, mà là giữ được những thay đổi khách hàng thực sự muốn thực hiện.

Em đề xuất mình chia thành hai giai đoạn để vừa kiểm soát thời gian vừa giảm rủi ro:

**Giai đoạn 1:** Ưu tiên xử lý những trường hợp phổ biến như thêm mã, xóa mã và giữ nguyên các thay đổi đã thực hiện trên thiết bị khác. Đây là phần trực tiếp ngăn sự cố vừa xảy ra.

**Giai đoạn 2:** Hoàn thiện những trường hợp phức tạp hơn, chẳng hạn hai thiết bị cùng sửa ghi chú hoặc thay đổi thứ tự của một mã.
