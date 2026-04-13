# s05: Skills

`s01 > s02 > s03 > s04 > [ s05 ] > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì
- Tại sao nhồi nhét tất cả kiến thức miền vào System Prompt lãng phí Token
- Mẫu tải hai tầng: tên rẻ tiền ở phía trước, nội dung đắt tiền theo yêu cầu
- Cách frontmatter (metadata YAML ở đầu file) đặt tên và mô tả cho mỗi kỹ năng
- Cách mô hình tự quyết định kỹ năng nào cần tải và khi nào

Bạn không thuộc lòng mọi công thức trong mọi cuốn sách nấu ăn bạn có. Bạn biết cuốn nào nằm ở kệ nào, và bạn chỉ lấy một cuốn xuống khi bạn thực sự nấu món đó. Kiến thức miền của một Agent hoạt động theo cách tương tự. Bạn có thể có các file chuyên môn về quy trình git, các mẫu kiểm thử, danh sách kiểm tra code review, xử lý PDF -- hàng chục chủ đề. Tải tất cả chúng vào System Prompt cho mỗi yêu cầu giống như đọc mọi cuốn sách nấu ăn từ đầu đến cuối trước khi đập quả trứng đầu tiên. Hầu hết kiến thức đó không liên quan đến bất kỳ tác vụ cụ thể nào.

## Vấn Đề

Bạn muốn Agent của mình tuân theo các quy trình làm việc miền cụ thể: quy ước git, các thực hành kiểm thử tốt nhất, danh sách kiểm tra code review. Cách tiếp cận đơn giản nhất là đưa mọi thứ vào System Prompt. Nhưng 10 kỹ năng với 2.000 Token mỗi kỹ năng nghĩa là 20.000 Token hướng dẫn cho mỗi lần gọi API -- hầu hết không liên quan gì đến câu hỏi hiện tại. Bạn trả tiền cho những Token đó mỗi lượt, và tệ hơn, tất cả văn bản không liên quan đó cạnh tranh sự chú ý của mô hình với nội dung thực sự quan trọng.

## Giải Pháp

Tách kiến thức thành hai tầng. Tầng 1 nằm trong System Prompt và rẻ: chỉ là tên kỹ năng và mô tả một dòng (~100 Token mỗi kỹ năng). Tầng 2 là nội dung kỹ năng đầy đủ, được tải theo yêu cầu thông qua lần gọi Tool chỉ khi mô hình quyết định nó cần kiến thức đó.

```
System prompt (Layer 1 -- always present):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

When model calls load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 -- on demand):  |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

## Cách Hoạt Động

**Bước 1.** Mỗi kỹ năng là một thư mục chứa file `SKILL.md`. File bắt đầu bằng frontmatter YAML (một khối metadata được phân cách bởi các dòng `---`) khai báo tên và mô tả của kỹ năng, theo sau là nội dung hướng dẫn đầy đủ.

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

**Bước 2.** `SkillLoader` quét tất cả các file `SKILL.md` khi khởi động. Nó phân tích frontmatter để trích xuất tên và mô tả, và lưu trữ nội dung đầy đủ để truy xuất sau.

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            # Use the frontmatter name, or fall back to the directory name
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        """Layer 1: cheap one-liners for the system prompt."""
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        """Layer 2: full body, returned as a tool_result."""
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

**Bước 3.** Tầng 1 được đưa vào System Prompt để mô hình luôn biết có những kỹ năng gì. Tầng 2 được kết nối như một hàm xử lý Tool bình thường -- mô hình gọi `load_skill` khi nó quyết định cần hướng dẫn đầy đủ.

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

Mô hình học được những kỹ năng nào tồn tại (rẻ, ~100 Token mỗi kỹ năng) và tải chúng chỉ khi liên quan (đắt, ~2000 Token mỗi kỹ năng). Trong một lượt thông thường, chỉ có một kỹ năng được tải thay vì tất cả mười.

## Những Gì Đã Thay Đổi So Với s04

| Thành phần     | Trước đây (s04)  | Sau này (s05)              |
|----------------|------------------|----------------------------|
| Tools          | 5 (cơ bản + task)| 5 (cơ bản + load_skill)    |
| System Prompt  | Chuỗi tĩnh       | + mô tả kỹ năng            |
| Kiến thức      | Không có         | Các file skills/\*/SKILL.md|
| Injection      | Không có         | Hai tầng (system + result) |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Giải thích tại sao "liệt kê trước, tải sau" tốt hơn nhồi nhét mọi thứ vào System Prompt
- Viết một file `SKILL.md` với frontmatter YAML mà `SkillLoader` có thể khám phá
- Kết nối tải hai tầng: mô tả rẻ trong System Prompt, nội dung đầy đủ qua `tool_result`
- Để mô hình tự quyết định khi nào kiến thức miền đáng được tải

Bạn chưa cần hệ thống xếp hạng kỹ năng, hợp nhất đa nhà cung cấp, các mẫu có tham số, hay các quy tắc khôi phục. Mẫu cốt lõi rất đơn giản: quảng cáo rẻ tiền, tải theo yêu cầu.

## Tiếp Theo Là Gì

Bây giờ bạn đã biết cách giữ kiến thức ngoài Context cho đến khi cần thiết. Nhưng điều gì xảy ra khi Context vẫn tăng lên -- sau hàng chục lượt làm việc thực tế? Trong s06, bạn sẽ học cách nén một cuộc hội thoại dài xuống còn những gì cốt yếu để Agent có thể tiếp tục làm việc mà không bị chạm ngưỡng Token.

## Điểm Mấu Chốt

> Quảng cáo tên kỹ năng một cách rẻ tiền trong System Prompt; tải nội dung đầy đủ thông qua lần gọi Tool chỉ khi mô hình thực sự cần.
