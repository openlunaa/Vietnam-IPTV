# Vietnam IPTV

Thư viện các kênh truyền hình trực tuyến Việt Nam cho nền tảng OpenLunaAI Streams.

## Giới thiệu

Vietnam IPTV là một phần của dự án OpenLunaAI, cung cấp khả năng xem các kênh truyền hình Việt Nam trực tiếp thông qua giao thức Internet. Dự án này tích hợp nhiều nguồn nội dung truyền hình và video từ nhiều nhà cung cấp, bao gồm:

- Danh sách kênh IPTV từ iptv-org
- TV360
- VieON
- SCTV
- Các nguồn nội dung Việt Nam khác

## Tính năng

- **Danh sách kênh đa dạng**: Các kênh truyền hình Việt Nam từ nhiều nhà cung cấp
- **Phân loại kênh theo thể loại**: Tin tức, giải trí, thể thao, giáo dục, thiếu nhi...
- **Trình phát đa định dạng**: Hỗ trợ HLS (m3u8), iframe embed, và nhiều định dạng khác
- **Tìm kiếm và lọc kênh**: Dễ dàng tìm kiếm kênh theo tên hoặc lọc theo thể loại
- **Thông tin kênh chi tiết**: Tên, logo, mô tả, ngôn ngữ, và thông tin khác
- **Truy cập không giới hạn**: Kết hợp với VPN để xem nội dung bị giới hạn địa lý

## Cài đặt

### Yêu cầu

- Node.js (phiên bản 18+)
- Package manager (npm, yarn, hoặc pnpm)
- Các API key cần thiết (TV360, VieON - tùy chọn)

### Cài đặt từ nguồn

```bash
# Clone repository
git clone https://github.com/openlunaa/Vietnam-IPTV.git
cd Vietnam-IPTV

# Cài đặt các dependency
npm install

# Khởi chạy ứng dụng
npm run dev
```

## Cấu trúc dự án

```
vietnam-iptv/
├── client/               # Mã nguồn frontend
│   ├── components/       # Các thành phần UI
│   │   ├── IPTVPlayer.tsx      # Trình phát video
│   │   └── ...
│   ├── pages/            # Các trang
│   │   └── VietnamIPTVPage.tsx # Trang chính hiển thị kênh Việt Nam
│   └── ...
├── server/               # Mã nguồn backend
│   ├── routes/           # Các route API
│   │   └── vietnam-iptv-api.ts # API lấy kênh IPTV Việt Nam
│   └── ...
└── ...
```

## API Endpoints

### Lấy danh sách kênh IPTV Việt Nam

```
GET /api/iptv/vietnam
```

**Phản hồi:**
```json
[
  {
    "id": "vtv1",
    "name": "VTV1",
    "logo": "https://example.com/vtv1-logo.png",
    "group": "general",
    "source": "iptv-org",
    "url": "https://example.com/vtv1-stream.m3u8",
    "isLive": true,
    "language": "Vietnamese",
    "country": "Vietnam"
  },
  ...
]
```

## Nguồn dữ liệu

Dự án sử dụng nhiều nguồn dữ liệu khác nhau:

1. **iptv-org**: Danh sách kênh IPTV toàn cầu mã nguồn mở (https://github.com/iptv-org/iptv)
2. **TV360**: Dịch vụ truyền hình trực tuyến tại Việt Nam
3. **VieON**: Nền tảng xem phim, TV show và các kênh truyền hình
4. **SCTV**: Truyền hình cáp Saigontourist
5. **Các nguồn nội dung khác**: XemTVTrucTuyen và các nền tảng khác

## Đóng góp

Chúng tôi hoan nghênh mọi đóng góp để cải thiện dự án Vietnam IPTV! Vui lòng làm theo các bước sau:

1. Fork repository
2. Tạo branch mới cho tính năng hoặc bản sửa lỗi của bạn
3. Thêm hoặc chỉnh sửa mã nguồn
4. Tạo Pull Request

## Vấn đề pháp lý

Dự án này chỉ định tuyến người dùng đến các nguồn nội dung có sẵn công khai trên Internet. Chúng tôi không lưu trữ bất kỳ nội dung truyền hình hoặc video nào. Dự án chỉ nhằm mục đích giáo dục và cá nhân.

## Giấy phép

Dự án này được phân phối theo giấy phép MIT. Xem file `LICENSE` để biết thêm thông tin.

## Liên hệ

Website: [openlunaai.net](https://openlunaai.net)

GitHub: [github.com/openlunaa](https://github.com/openlunaa)