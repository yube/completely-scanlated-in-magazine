from io import BytesIO
import requests
from PIL import Image, ImageDraw, ImageFont
from concurrent.futures import ThreadPoolExecutor
from tqdm import tqdm
import threading

mag_name = 'Comic LO'

def resize_image(img, max_height=218):
    width, height = img.size
    if height > max_height:
        new_height = max_height
        aspect_ratio = width / height
        new_width = int(new_height * aspect_ratio)
        img = img.resize((new_width, new_height), Image.LANCZOS)
    return img

def break_text(text, max_length=18):
    if len(text) <= max_length:
        return [text]
    lines = []
    current_line = ""
    for word in text.split(" "):
        if len(current_line) + len(word) + 1 > max_length:
            lines.append(current_line)
            current_line = ""
        current_line += (word + " ")
    lines.append(current_line)
    return lines

def truncate_text(text, max_length=50):
    return text[:45] + "..." if len(text) > max_length else text

def create_montage(images, titles, images_per_row=10):
    images = [img for img in images if img is not None]
    if len(images) == 0:
        print("No images to create a montage.")
        return

    img_width = max(img.size[0] for img in images)
    img_height = max(img.size[1] for img in images)
    text_height = 30
    title_height = 50
    new_img_height = img_height + text_height

    num_rows = (len(images) - 1) // images_per_row + 1
    montage_width = img_width * min(images_per_row, len(images))
    montage_height = new_img_height * num_rows + title_height

    montage = Image.new(mode="RGB", size=(montage_width, montage_height), color=(255, 255, 255))
    draw = ImageDraw.Draw(montage)

    try:
        font = ImageFont.truetype("arial.ttf", 16)
        title_font = ImageFont.truetype("arial.ttf", 24)
    except IOError:
        font = ImageFont.load_default()
        title_font = ImageFont.load_default()

    title_text = f"Completely scanlated manga in {mag_name}"

    bbox = draw.textbbox((0, 0), title_text, font=title_font)
    title_width = bbox[2] - bbox[0]
    title_height = bbox[3] - bbox[1]

    title_position = ((montage_width - title_width) // 2, 0)
    draw.text(title_position, title_text, font=title_font, fill=(0, 0, 0))

    for i, (img, title) in enumerate(zip(images, titles)):
        row = i // images_per_row
        col = i % images_per_row
        x_offset = col * img_width
        y_offset = row * new_img_height + title_height

        montage.paste(img, (x_offset, y_offset))

        truncated_title = truncate_text(title)
        lines = break_text(truncated_title)
        bbox = font.getbbox("A")
        line_height = bbox[3] - bbox[1]
        line_spacing = line_height + 2

        for j, line in enumerate(lines):
            text_y = y_offset + img_height + j * line_spacing
            draw.text((x_offset, text_y), line.strip(), font=font, fill=(0, 0, 0))

    montage.save(f"{mag_name}.png")
    print(f"Chart saved as {mag_name}.png")


def get_info(id):
    get_url = f'https://api.mangaupdates.com/v1/series/{id}'
    response = requests.get(get_url)
    data = response.json()
    title = data.get('title')
    url = data.get('url')
    image = data.get('image').get('url').get('original')
    completed = data.get('completed')
    status = data.get('status')
    if completed and 'Oneshot' not in status:
        info.append({
            'title': title,
            'url': url,
            'image': image,
            'completed': completed
        })

        print(f'{title}: {url} - Completed - {status}')
    # else:
        # print(f'{title}: {url} - Unfinished o')

def get_mag(name):
    url = 'https://api.mangaupdates.com/v1/publishers/publication'

    params = {
        'pubname': name
    }

    headers = {
        "Content-Type": "application/json"
    }

    response = requests.get(url, headers=headers, params=params)

    try:
        result = response.json()
        # print(json.dumps(result, indent=2))

        ids = [series['series_id'] for series in result.get('series_list', [])]
        print(ids)
        return ids
    except Exception as e:
        print("Error:", e)
        print("Response:", response.text)

def prep_image(info_list):
    images = []
    titles = []
    for entry in info_list:
        try:
            response = requests.get(entry['image'])
            img = Image.open(BytesIO(response.content)).convert('RGB')
            resized = resize_image(img=img)
            images.append(resized)
            titles.append(entry['title'])
        except Exception as e:
            print(f"failed to load image for {entry['title']}:{e}")
            continue
    return images, titles

# info = []
# ids = get_mag(mag_name)
# for id in ids:
#     get_info(id)

info = []
ids = get_mag(mag_name)

lock = threading.Lock()

def safe_get_info(series_id):
    try:
        get_url = f'https://api.mangaupdates.com/v1/series/{series_id}'
        response = requests.get(get_url)
        data = response.json()
        title = data.get('title')
        url = data.get('url')
        image = data.get('image').get('url').get('original')
        completed = data.get('completed')
        status = data.get('status')

        if completed and 'Oneshot' not in status:
            with lock:
                info.append({
                    'title': title,
                    'url': url,
                    'image': image,
                    'completed': completed
                })
            print(f'{title}: {url} - Completed - {status}')
    except Exception as e:
        print(f"Error fetching series {series_id}: {e}")

# Parallelize
with ThreadPoolExecutor(max_workers=10) as executor:
    list(tqdm(executor.map(safe_get_info, ids), total=len(ids)))


images, titles = prep_image(info)
create_montage(images, titles)
