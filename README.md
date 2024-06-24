# Quản lý Auth trong Next.js

Để xác thực một request thì backend thường sẽ xác thực qua 2 cách:

1. FE gửi token qua header của request như `Authorization: Bearer <token>` (token thường được lưu trong localStorage của trình duyệt)
2. FE gửi token qua cookie của request (sự thật là cookie cũng nằm trong header của request)

Cách dùng Cookie có ưu điểm là an toàn hơn 1 chút so với cách dùng localStorage, nhưng đòi hỏi setup giữa Backend và FrontEnd phức tạp hơn.

Next.js chúng ta có thể dùng 2 cách trên, nhưng nó phức tạp hơn so với React.Js Client Side Rendering (CSR) truyền thống vì Next.js có cả Server và Client

## Cách 1: Dùng localStorage

Cách này chỉ áp dụng cho server check authentication dựa vào header `Authorization` của request.

- Tại trang login, chúng ta gọi api `/api/login` để đăng nhập. Nếu đăng nhập thành công, server sẽ trả về token, chúng ta lưu token vào localStorage. Việc này chúng ta sẽ làm ở phía client hoàn toàn.

- Tại những trang không cần authenticated, chúng ta có thể gọi api ở cả server và client của next.js mà không cần phải làm gì thêm.

Vấn đề sẽ nằm ở những trang cần authenticated. Làm sao để Next.js biết được user đã authenticated hay chưa? Để giải quyết vấn đề này chúng ta cần thiết kế một middleware

### Middleware ở Next.js

Middleware ở Next.js thì có 2 loại:

1. Middleware hoạt động ở client next (giống như những gì chúng ta đã làm trước đây ở React.js truyền thống)
2. Middleware hoạt động ở server next

#### Middleware ở client next

Nếu dùng middleware client thì chỉ cần tạo 1 `use client` `AuthenticatedComponent` và wrap nó ở những trang cần authenticated.

```tsx
'use client'
export default function AuthenticatedComponent({ children }) {
  const token = localStorage.getItem('token')
  if (!token) return <div>Chưa đăng nhập</div>
  return children
}
```

Cách dùng middleware này là server next.js sẽ không biết được user đã authenticated hay chưa. Ví dụ bạn truy cập vào trang `/profile` (cần authenticated) thì đây là flow diễn ra

Bạn enter url `/profile`
=> Trình duyệt gửi request đến server Next.js (request này sẽ gửi kèm cookie nếu có)
=> Server Next.js sẽ render trang `/profile` vì không biết được user đã authenticated hay chưa và trả về trình duyệt
=> Trình duyệt nhận được trang `/profile` và chạy `use client` `AuthenticatedComponent`
=> `AuthenticatedComponent` sẽ kiểm tra xem có token trong localStorage không, nếu có thì render trang `/profile` ra, nếu không thì render ra `Chưa đăng nhập`

Kết quả vẫn đúng, người dùng vẫn thấy trang `/profile` nếu đã authenticated nhưng cách này có một số khuyết điểm

- Profile Component phải là một client nếu chúng ta cần fetch các api cần authenticated, vì chỉ có client mới có thể truy cập được vào localStorage

- Không đồng nhất giữa server và client, điều này không tốt.

Cách giải quyết là dùng middleware ở server next.js

#### Middleware ở server next

Next.js cung cấp 1 cách để chúng ta có thể dùng middleware ở server next.js, có thể xem [tại đây](https://nextjs.org/docs/app/building-your-application/routing/middleware)

Middleware này sẽ chạy ngay khi có request gửi đến server Next.js, trước khi trang được render ở server.

Nhưng chúng ta cần 1 thứ gì đó để Next.js biết được user đã authenticated hay chưa, và thứ đó là chỉ có thể là cookie từ trình duyệt gửi lên. Vì khi bạn enter url `/profile` thì chỉ có cookie là được gửi kèm theo request đến server Next.js.

Nãy giờ chưa setup cookie gì cả, bây giờ chúng ta sẽ setup logic cookie. Đó là khi chúng ta login thành công thì chúng ta sẽ set cookie là `isLogged=true` vào trình duyệt ở client luôn. Cookie này có thời hạn tương tự với token, và cookie `isLogged` có thể dùng JavaScript can thiệp được. Như vậy thì khi request đến server Next.js thì server sẽ biết được user đã authenticated hay chưa dựa vào cookie `isLogged`. Client next.js cũng sẽ biết được user đã authenticated hay chưa dựa vào cookie `isLogged` (hoặc giá trị lưu trong localStorage tùy thích, nhưng khuyến khích dùng `isLogged` từ cookie cho đồng bộ).

Và đây là middleware ở server next.js

```tsx
export const config = {
  matcher: ['/profile']
}
export function middleware(request: NextRequest) {
  const isLogged =
    (request.cookies.get('isLogged')?.value as string | undefined) === 'true'
  if (!isLogged) return new Response('Chưa đăng nhập', { status: 401 })
}
```

Ưu điểm cách này là đồng bộ được giữa server và client.

### Gọi api trong next.js

Xong phần middleware cho localStorage, giờ chúng ta sẽ tìm hiểu cách gọi api trong next.js

Gọi API thì cũng có 2 cách là gọi ở client và gọi ở server. Ở đây mình chỉ bàn về việc gọi các API cần authenticated, vì những API không cần authenticated thì gọi ở cả client và server đều được.

Nếu gọi API cần authenticated như GET `/api/profile` thì chúng ta chỉ cần gán token vào header `Authorization` là xong. Y hệt như gọi API ở React.js truyền thống.

Còn gọi API cần authenticated ở server next.js thì làm thế nào để gán được token vào header `Authorization`, vì ở server Next.js, bạn không thể truy cập vào được localStorage của trình duyệt.

Thực sự đây chính là khuyết điểm của việc dùng localStorage để Authentication với Next.js.

Dù sao thì các route cần authenticated cũng không cần SEO nên không cần gọi ở server để SEO làm gì cả. Bạn hoàn toàn có thể gọi api ở client, nếu bạn chấp nhận điều này thì không sao cả.

Nhưng với cá nhân mình là người cầu toàn thì không thích khuyết điểm này lắm, chưa kể là Next.js với tôn chỉ là ưu tiên mọi thứ ở server.

Để giải quyết điều này thì chúng ta không nên dùng LocalSoage mà nên dùng Cookie để lưu token nhé. Đi đến cách 2 nào.

## Cách 2: Dùng Cookie

Cách này áp dụng cho Server check token dựa vào cookie hay header `Authorization` đều được.

Tại trang login chúng ta gọi api là `/app/login` từ Server Action để đăng nhập. Chúng ta dùng Server Action để làm proxy, trong server action, khi login thành công, chúng ta sẽ set cookie `token` vào trình duyệt và trả về token cho client để client set vào Context API hoặc caching react tùy thích (phục vụ nếu cần gọi api ở client).

# Client component

## React SPA truyền thống (React Vite, CRA, ...) là 1 client component khổng lồ

Khi lần đầu vào 1 trang web

1. Trình duyệt **request** đến server và trả về file `index.html` cơ bản (hầu như không chứa html gì nhiều)
2. Trình duyệt nhận thấy trong file html có link đến file js, css nên là **request lần nữa** đến server để lấy file js, css
3. Trình duyệt tiến hành chạy code JS để render ra HTML và gắn sự kiện vào HTML đó
4. Người dùng thấy và tương tác được với trang web

Trong quá trình này, web sẽ trắng xóa cho đến khi bước thứ 3 được hoàn thành.

Vậy nên mới nói lần đầu tiên khi truy cập vào các SPA truyền thống khá lâu, nhưng sau đó thì thao tác hay chuyển trang sẽ rất nhanh vì js bundle cả app đã có ở client rồi, nếu cần data thì mới request đến server lấy data thôi.

Các bạn để ý cái bước thứ 3, lúc nào HTML cũng được JavaScript trình duyệt render ra khi chúng ta truy cập vào web. Cái này gọi là **Dynamic Rendering**

Với Dynamic Rendering, HTML được render ra khi chúng ta request, có thể được render ở client hoặc server đều được.

## Client Component Next.js

Dùng client component khi:

- Cần tương tác: dùng hook, useState, useEffect, event listener (onClick, onSubmit, onChange,...), ...
- Cần dùng các API từ trình duyệt

Trong Next.js, mặc định tất cả các component đều được render ra HTML sẵn khi có thể lúc Nextjs build (Static Rendering). Kể cả Server component và Client component.

Vậy nên khi bạn truy cập vào 1 trang web Next.js, bạn sẽ thấy UI ngay lập tức do Server Next.js trả về HTML đã render sẵn. Sau đó trình duyệt sẽ render lại CLient Component 1 lần nữa để đồng bộ DOM, sự kiện, state, effect.

Rút ra được điều gì từ đây?

- Client Component bị render tối thiểu 2 lần: 1 lần khi build, 1+ lần ở client
- Vì trả về HTML sẵn nên người dùng có thể thấy content ngay lập tức (Tăng UX)
- Dù thấy content ngay lập tức nhưng vẫn không thể tương tác ngay được vì cần phải chờ trình duyệt đồng bộ lại client component (render, gắn sự kiện, state, effect...)

Ưu điểm của Client Component:

- Giảm gánh nặng cho server khi component nặng và phức tạp về logic => Server yếu thì nên dùng

Nhược điểm của Client Component:

- SEO không tốt
- Thiết bị client yếu thì chạy không nổi
- Tăng bundle size javascript

Lời khuyên từ cá nhân Được:

Dùng Server Component khi có thể, Được không đặt nặng vấn đề về cấu hình Server, vì dùng cho production thì server phải tốt. Quan trọng là trải nghiệm người dùng

# 6 | CSS trong Next.js

## Global style

Khi cần thêm CSS cho cả app: Ví dụ các thẻ cơ bản `body, html, a, p, h1, h2, h3, h4, h5, h6, ...` hay `*`, hoặc đôi khi cần thêm một số class để dùng toàn app thì cũng có thể thêm ở đây

- CSS ở file `src/app/globals.css`
- Nếu dùng tailwind thì nên dùng `@layer` để đảm bảo về tính dễ đọc cũng như là độ ưu tiên css khi build

> Lưu ý rằng file này chỉ import 1 lần duy nhất trong toàn app

## Tạo 1 class css phức tạp mà tailwind không hỗ trợ hoặc override 1 class thư viện nào đó

- Dùng CSS Module để đảm bảo không bị xung đột với class css khác

## Khi cần toggle class hoặc css động

- Dùng `clsx`

## Khác

Ngoài ra còn 1 số giải pháp khác như styled component, emotion, styled-jsx,... Nhưng ở trên là đủ dùng và best practice cho 1 app Next.js thông thường

# Cơ chế rendering

Có 2 môi trường mà web chúng ta có thể render

Client: đại diện trình duyệt người dùng
Server: đại diện cho máy chủ nơi chứa data và trả về response

Client và Server là 2 môi trường tách biệt với nhau. Đây gọi là **Network Boundary**

Vì next.js có khả năng render code React ở server và client nên đôi khi dev hiểu nhầm rằng 2 môi trường là một.

Với Next.js, code lúc nào cũng phải phân biệt rõ ràng giữa 2 môi trường này bằng từ khóa `'use client'` hoặc `'use server'`

Ví dụ đang ở môi trường client, muốn truy cập data ở server thì cần phải gửi 1 request mới đến server mới lấy được.

# Setup môi trường

## Bắt buộc

- Cài Node.js (ưu tiên dùng NVM để dễ dàng chuyển đổi version): > 18.17
- Hệ điều hành: Windows, MacOS, Linux đều được
- Cài Git để quản lý source code

2 cái trên thì ai học React.js hay Node.js cũng có rồi, mình chỉ nhắc lại

## Tùy chọn

Đây là setup máy mình, bạn có thể tham khảo và tùy chỉnh theo ý

- Trình duyệt Chrome
- IDE: Visual Studio Code với theme Dracular và font Cascadia Code tích hợp ligature, mua gói Copilot nếu có điều kiện
- Setup VS Code: [Cách mình setup VS Code | Extensions, Themes, Setting, Tips và Tricks](https://duthanhduoc.com/blog/cach-minh-setup-vs-code)
- Setup Macbook: [Cách mình setup Macbook để code](https://duthanhduoc.com/blog/cach-minh-setup-macbook-de-code)

# Giới thiệu về Next.js

## 1. Next.js là gì?

- Next.js là fullstack framework cho React.js được tạo ra bởi Vercel (trước đây là ZEIT).
- Next có thể làm server như Express.js bên Node.js và có thể làm client như React.js

## 2. Next.js giải quyết vấn đề gì?

### Đầu tiên là render website ở Server nên thân thiện với SEO

React.js thuần chỉ là client side rendering, nhanh thì cũng có nhanh nhưng không tốt cho SEO. Ai nói với bạn rằng sài React.js thuần vẫn lên được top google ở nhiều thì đó là lừa đảo (hoặc họ chỉ đang nói 1 nữa sự thật)

Next.js hỗ trợ server side rendering, nghĩa là khi người dùng request lên server thì server sẽ render ra html rồi trả về cho người dùng. Điều này giúp cho SEO tốt hơn.

### Tích hợp nhiều tool mà React.js thuần không có

- Tối ưu image, font, script
- CSS module
- Routing
- Middleware
- Server Action
- SEO ...

### Thống nhất về cách viết code

Ở React.js, có quá nhiều cách viết code và không có quy chuẩn.

Ví dụ:

- Routing có thể dùng React Router Dom hoặc TanStack Router.
- Nhiều cách bố trí thư mục khác nhau

Dẫn đến sự không đồng đều khi làm việc nhóm và khó bảo trì.

Next.js giúp bạn thống nhất về cách viết code theo chuẩn của họ => giải quyết phần nào đó các vấn đề trên

### Đem tiền về cho Vercel 🙃

Ngày xưa các website thường đi theo hướng Server Side Rendering kiểu Multi Page Application (MPA) như PHP, Ruby on Rails, Django, Express.js ... Ưu điểm là web load nhanh và SEO tốt, nhưng nhược điểm là UX hay bị chớp chớp khi chuyển trang và khó làm các logic phức tạp bên client.

Sau đó React.js, Angular, Vue ra đời, đi theo hướng Single Page Application (SPA) giải quyết được nhược điểm của MPA, nhưng lại tạo ra nhược điểm mới là SEO kém và load chậm ở lần đầu.

Vercel là công ty cung cấp các dịch vụ phía Server như hosting website, serverless function, database, ...và họ cũng là công ty đầu tiên khởi xướng trào lưu "quay trở về Server Side Rendering" .

Vì thế họ tạo ra Next.js, vừa để khắc phục nhược điểm của SPA truyền thống, vừa gián tiếp bán các sản phẩm dịch vụ của họ. Ví dụ Next.js chạy trên dịch vụ Edge Runtime của họ sẽ có độ trễ thấp hơn so với chạy trên Node.js

## 3. Yêu cầu khi học Next.js

- Cần biết HTML, CSS, JavaScript
- Cần biết React.js cơ bản (Thao khảo khóa học [React.js Super](https://duthanhduoc.com/courses/react))
- Cần biết Node.js cơ bản(Thao khảo khóa học [Node.js Super](https://duthanhduoc.com/courses/nodejs-super))

## FAQ

1. Có nên dùng Next.js làm Backend luôn không?

Nếu bạn cần làm 1 dự án nhỏ cỡ 1-5 người làm, thời gian triển khai nhanh, không yêu cầu nhiều nghiệp vụ phức tạp thì có thể dùng Next.js làm fullstack framework luôn

Còn lại thì chỉ nên dùng Next.js làm Front-End thôi. Vì backend Next.js sẽ thiếu nhiều tính năng hơn khi so sánh với các framework chuyên backend khác. Chưa hết, dùng Next.js làm backend bạn sẽ dính vào hệ sinh thái Node.js

2. Làm website quản lý không cần SEO thì có nên dùng Next.js không?

Không cần thiết, có thể dùng React.js Vite truyền thống.

Nếu bạn sợ trong tương lai có làm mấy cái landing page hay trang public ra ngoài thì chọn Next.js là lựa chọn an toàn.

3. Next.js có phù hợp với dự án lớn không?

Có. Rất nhiều dự án lớn dùng Next.js như Tiktok, Netflix, Uber, ...

4. Next.js deploy ở đâu?

Nên deploy trên VPS (tức là máy chủ ảo)

Ngoài ra có thể deploy trên Vercel, Netlify. Nếu free thì chậm (phù hợp demo), còn trả phí thì khá là đắt.

5. Khóa học này dạy App Router hay Page Router?

App Router, vì nó đã ra đời hơn 1 năm rồi và ổn định. Nó là tương lai của Next.js

# Next.js render component của bạn như thế nào?

Component ở đây bao gồm Server Component và Client Component

## Khi chúng ta build

Mọi component dù là Server Component hay Client Component khi build đều sẽ có

- Static HTML
- JS Bundle
- Ngoài ra còn có CSS Bundle, Image, Font,...

## Khi request lần đầu tiên (full page load)

1. Server Next.Js render server component và kết hợp với Client Component để tạo ra HTML để gửi về client

2. Client ngay lập tức thấy được website nhưng chưa tương tác được với nó (ví dụ chưa click, hover,...)

3. Trong đống JS Bundle download về có chứa **React Server Component Payload (RSC Payload)**, cái này dùng để để render lại client component ở client, cập nhật DOM

4. Cuối cùng là sẽ thêm các sự kiện vào các client component để tương tác với người dùng => Bước này gọi là Hydration, sau bước này thì có thể tương tác với website

> React Server Component Payload là 1 data đặc biệt được render ở phía Server phục vụ cho việc đồng bộ, cập nhật DOM giữa Client Component và Server Component

## Khi request lần thứ 2 (Subsequent Navigations)

Ví dụ chúng ta navigate từ `/home` sang `/about`

Thì server Next.js sẽ không trả HTML về cho chúng ta nữa mà trả React Server Component Payload (RSC Payload) và các bundle JS, CSS cần thiết.

Client sẽ tự render ra HTML

Điều này sẽ giúp việc navigation nhanh hơn, nhưng vẫn đảm bảo về SEO

# Server Component

Đây là chế độ mặc định của component trong Next.js

Ưu điểm:

- Fetch data ở server => Nơi gần data center nên là sẽ nhanh hơn là fetch ở client => Giảm thiểu thời gian rendering, tăng UX
- Bảo mật: Server cho phép giữ các data nhạy cảm, logic đặc biệt không muốn public ở client
- Caching: Vì được render ở server nên có thể lưu giữ cache cho nhiều người dùng khác nhau => Không cần render trên mỗi request
- Bundle Size: Giảm thiểu JS bundle size vì client không cần tải về phần JS logic để render HTML
- Load trang lần đầu nhanh và chỉ số FCP (First Contentful Paint) thấp do người dùng sẽ thấy content ngay lập tức
- Search Engine Optimization and Social Network Shareability
- Streaming

=> Ưu tiên dùng Server Component khi có thể

# Một số vấn đề chưa giải quyết

- Video này không có trong dự định, để thực hiện video này mình phải chỉnh sửa lại hệ thống authentication backend, nhưng để đảm bảo tính tương thích ngược (để các bạn sau xem các video trước đó làm vẫn được) thì mình không thể thay đổi hết hệ thống authentication backend của mình được.

- Chúng ta dùng session token để lưu giữ phiên đăng nhập, session token của mình là 1 JWT có `exp` là thời gian hết hạn của nó. Nếu backend thay đổi thời gian hết hạn token thì JWT sẽ là một JWT mới, điều này không đúng theo concept session token. Vì nếu dùng session token chúng ta chỉ cần thay đổi thời gian hạn của token thôi, chứ không phải tạo ra một token mới.

- Vậy nên khi xem video này, các bạn có thể bỏ qua giá trị `exp` trong JWT, **coi như nó vô nghĩa**. Chúng ta sẽ dùng một giá trị khác để xác định thời gian hết hạn của token.

## Khi đang dùng mà session token hết hạn thì sao?

Thì phải cho user đăng xuất.

Nhưng nếu đang thực hiện chức năng quan trọng mà bắt user đăng xuất => không tốt về mặt UX

Cách tốt nhất để giải quyết là trong lúc người dùng đang dùng web thì chúng ta tăng thời gian hết hạn của session

Để làm được thì cần 2 yếu tố:

- Backend của bạn phải hỗ trợ chức năng Sliding Session, tức là tăng giá trị expire của session

- Frontend của bạn phải kiểm tra thời gian hết hạn của session token và tăng thời gian hết hạn của nó trước khi nó hết hạn. Vì session token hết hạn thì coi như vô dụng. Vậy nên cần refresh trước khi nó hết hạn

Ví dụ session token hết hạn sau 15 ngày thì mỗi khi thời hạn hết hạn còn dưới 7 ngày refresh lại một lần.

Trong trường hợp người ta không mở website 15 ngày thì khi mở lên sẽ bị đăng xuất

Cách làm này gần giống với phương pháp refresh token, chỉ khác là khi refresh token chúng ta nhận lại cặp access token và refresh token mới. Còn khi dùng session này thì token vẫn giữ nguyên, chỉ là thời gian hết hạn của nó được tăng lên

## Nếu tôi dùng access token và refresh token thì sao?

## Dùng axios thì sa?
