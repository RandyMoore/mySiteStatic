---
title: "Top 20 Skills Over Time - remoteok.io"
date: 2020-08-08T00:00:00+00:00
draft: false
---
As a professional developer you must always be learning. It can be frustrating to determine what skills are actually in demand. It helps to look at job postings but this takes a lot of time and you don't necessarily get a bird's eye view of how the demand for different skills varies. So this is a great opportunity to apply artificial intelligence and some simple data visualization to have a machine illustrate what skills are in demand, and how they change over time. In this case we visualize how the most mentioned skills change on [remoteok.io](https://remoteok.io) during the first half of 2020.

{{< youtube bkCAK_GeJeA >}}

### Legend
* \# Posts: Total number of posts on remoteok.io that day
* Total # NEs: The sum of counts of the top 20 Named Entities (skills)

The y-axis has a 10% marking. The bar for each skill is scaled according to what % of the Total # NEs count it represents.

# How? (Summary)
1. Collect the raw data, in this case all the job postings on remoteok.io. Data was collected every couple weeks for a few months by my good friend Pi who spent a few mintues looking at every post before copying the text.
2. Use Natural Language Processing (NLP) to extract Named Entities from each post.
3. Filter out all Named Entities that are not a skill.
4. Take the top 20 skills by frequency for each day across all posts.
5. Use a blender script to generate video of the top 20 skills and illustrate how they change between the days.
6. Mix some music into the video so it is more entertaining.

# How? (Details)
This data analysis was done using only open source tools. It goes to show how powerful tools are readily available and easy to use, even in the hands of an aspiring data scientist.

## Collect The Raw Data
Make a good friend like I did with Pi who is willing to spend several hours browsing to each post and copy the text into a file for each day. Have them be nice to whatever site you are collecting data from. In this case all the posts for a day where put into a single file named for the day, the text of each post was separated by "++++++".

## Use NLP to Extract Named Entities
[Spacey](https://spacy.io/) is used to perform the NLP processing. Here is the script which is hardcoded to open all the files named like "2020-07-12_remoteok.io.txt" in the current directory, with the date prefix different for each day. This script will fully utilize all the processor cores you have on your machine, it took about 2 minutes to process 14 files (~9MB total of text) on my i7-5700HQ. You do have to review the results and add named entities to the ignore list that are not skills and also add mappings for skills that are synonyms. Before running install spacey and then download the medium size English neural network with 

`spacy download "en_core_web_md"`

```python
from collections import Counter, defaultdict
import csv
import os
from multiprocessing import Pool

from spacy import load

TOP_N = 20
nlp = load("en_core_web_md")
remote_ok_files = sorted(f for f in os.listdir('.') if 'remoteok' in f)
ignore = [ # named entity with substring match of any of these
    "daily",
    "millions", 
    "Formstack",
    "WordPress",
    # There are actually ~50, most of them excluded for brevity.
    ]

synonyms = {
    "Javascript": "JavaScript",    
    "Computer Science": "Comp. Sci",
}

file_to_posts = defaultdict(dict)

def url_to_post(file_name):
    postings = {}
    with open(file_name, 'r') as f:
        posting = ""
        url = None
        for line in f.readlines():
            if "++++++" not in line:
                if url:
                    posting += line
                else:
                    url = line
            else:
                postings[url] = posting
                posting = ""
                url = None
    return postings

def nlp_on_post(url_post_file_name):
    url, post, file_name = url_post_file_name
    doc = nlp(post)
    return [synonyms.get(e.text, e.text) for e in doc.ents if not any(to_ignore in e.text for to_ignore in ignore)]

with open(f'top_{TOP_N}_skills.csv', 'w') as csv_out_file:
    csv_writer = csv.DictWriter(csv_out_file, fieldnames=['Day', 'Posts', 'Skill', 'Count'])
    csv_writer.writeheader()    
    for name in remote_ok_files:        
        url_to_posting = url_to_post(name)
        print(f"Processing {name}")
        with Pool() as po:
            named_entities_per_post = po.map_async(nlp_on_post, ((url, post, name) for url, post in url_to_posting.items())).get()
        for url_and_named_entities in zip(url_to_posting.keys(), named_entities_per_post):
            url, named_entities = url_and_named_entities
            file_to_posts[name][url] = named_entities    
        for ne, count in Counter(sorted(e for e_list in file_to_posts[name].values() for e in e_list)).most_common(TOP_N):
            csv_writer.writerow({'Day': name.split('_')[0], 'Posts': len(url_to_posting), 'Skill': ne, 'Count': count})        

```

This script will output a file named "top_20_skills.csv" with columns [Day, Posts, Skill, Count] which are the day (prefix of an input file), how many posts there were for that day, the skill, and the frequency of that skill for all the posts in the day.

## Use Blender to Visualize Data
[Blender](https://www.blender.org/) is an enormously powerful open source 3D rendering tool. It has many features including the ability to script actions that would otherwise take an infeasible amount of time to do manually. This script was developed and run on the latest Blender version 2.83 to produce the video. Just start a new Blender session, go to the scripting workspace, create a new script, copy and paste the script text, and run.
```python
import bpy
from collections import defaultdict
import csv
from datetime import date, timedelta
from random import random

TOP_N = 20
bar_spacing = 0.5
offscreen_y = -10
offscreen_x = TOP_N * (1 + bar_spacing) / 2

cur_frame = 1
frames_to_animate = 30
frames_to_pause = 60

# Read input file into local memory
day_to_skill_to_count = defaultdict(dict)
day_to_post_count = {}
with open(f'top_{TOP_N}_skills.csv', 'r') as csv_in:
    reader = csv.DictReader(csv_in)
    for row in reader:
        day_to_skill_to_count[row['Day']].update({row['Skill']: row['Count']})
        day_to_post_count[row['Day']] = row['Posts']

# Delete any existing objects
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete(use_global=False, confirm=False)

# Set up world, camera, light
bpy.data.worlds["World"].node_tree.nodes["Background"].inputs[0].default_value = (0, 0, 0, 1)
bpy.ops.object.light_add(type='AREA', radius=10, align='WORLD', location=(14.3, 7.6, 10))
bpy.context.object.data.energy = 10000
bpy.ops.object.camera_add(enter_editmode=False, align='VIEW', location=(14, 4.77, 10), rotation=(0, 0, 0))
bpy.context.object.data.lens = 11
bpy.context.scene.render.resolution_x = 1700
bpy.context.scene.render.resolution_y = 1200
    
# Generate the text and a vertical bar of random color for each of the Top N skills
skill_label_material = bpy.data.materials.new(name=f"Skill Label Material")
skill_label_material.diffuse_color = (.9, .8, .8, 1)

skill_to_bar_and_text = {}
for day in day_to_skill_to_count.keys():    
    for skill in day_to_skill_to_count[day].keys():
        if skill in skill_to_bar_and_text:
            continue
        print(f"Creating bar for {skill}")
        bpy.ops.mesh.primitive_plane_add(size=1)
        for v in bpy.context.object.data.vertices:
            # Translate so origin of bar is at bottom left, all scaling happens in +x, +y direction
            v.co[0] += 0.5
            v.co[1] += 0.5
        new_bar = bpy.context.object
        new_bar.data.materials.append(bpy.data.materials.new(name=f"Material for {new_bar.name}"))
        new_bar.data.materials[0].diffuse_color = (random(), random(), random(), 1)
        bpy.ops.object.text_add()
        new_text = bpy.context.object
        new_text.data.body = skill
        new_text.data.materials.append(skill_label_material)
        new_text.data.align_x = 'RIGHT'
        new_text.data.align_y = 'CENTER'
        bpy.ops.transform.rotate(value=1.5708)
        skill_to_bar_and_text[skill] = (new_bar, new_text)
        new_bar.location = (offscreen_x, offscreen_y, 0)
        new_text.location = (offscreen_x + 0.5, offscreen_y, 0)
        bpy.ops.object.select_all(action='DESELECT')

# Add y-axis label
white_material = bpy.data.materials.new(name=f"White Material")
white_material.diffuse_color = (1, 1, 1, 1)

bpy.ops.object.text_add()
y_axis_label = bpy.context.object
bpy.ops.transform.rotate(value=1.5708)
y_axis_label.data.align_y = 'BOTTOM'  
y_axis_label.data.body = "% of NE Total Count"
y_axis_label.location = (0, 0.5, 0)
y_axis_label.data.materials.append(white_material)
bpy.ops.object.text_add()
ten_percent_text = bpy.context.object
ten_percent_text.data.align_x = 'RIGHT'
ten_percent_text.data.align_y = 'CENTER'
ten_percent_text.data.body = "10%-"
ten_percent_text.location = (-0.2, 10, 0)
ten_percent_text.data.materials.append(white_material)
bpy.ops.object.select_all(action='DESELECT')
        
# Animate the bars and text across days to show how they change
all_bars_and_texts = set()
for bar, text in skill_to_bar_and_text.values():
    all_bars_and_texts |= {bar, text}
        
for day in sorted(day_to_skill_to_count.keys()):
    visible_bars_and_texts = set()
    skills_and_counts = sorted(list(day_to_skill_to_count[day].items()), key=lambda t: int(t[1]), reverse=True)
    total_skills_count = sum(int(t[1]) for t in skills_and_counts)
    for i, skill_count in enumerate(skills_and_counts):
        skill, count = skill_count
        count_scale = 100 * (int(count) / total_skills_count)
        print(f"Placing slot {i}: {skill} with {count}")
        bar, text = skill_to_bar_and_text[skill]
        bar.location = (i*(1+bar_spacing), 0, 0)
        bar.scale = (1, count_scale, 1)
        bar.keyframe_insert(data_path='location', frame=cur_frame)
        bar.keyframe_insert(data_path='scale', frame=cur_frame)
        text.location = (i*(1+bar_spacing) + 0.5, -0.5, 0)
        text.keyframe_insert(data_path='location', frame=cur_frame)
        visible_bars_and_texts |= {bar, text}
    for obj in all_bars_and_texts - visible_bars_and_texts:
        obj.location = (offscreen_x + (0.5 if obj.type == 'FONT' else 0), offscreen_y, 0)
        obj.scale = (1, 1, 1)
        obj.keyframe_insert(data_path='location', frame=cur_frame)
        obj.keyframe_insert(data_path='scale', frame=cur_frame)            
    cur_frame += frames_to_pause
    for obj in all_bars_and_texts:            
        obj.keyframe_insert(data_path='location', frame=cur_frame)
        obj.keyframe_insert(data_path='scale', frame=cur_frame)

    cur_frame += frames_to_animate

# Create the label for the Day, # of posts, total # of named entities
days_and_total_ne_count = [
    (day, sum(int(count) for count in skill_to_count.values())) 
    for day, skill_to_count in day_to_skill_to_count.items()]
bpy.ops.object.text_add()
day_text = bpy.context.object
day_text.location = (11.3, 11, 0)
day_text.data.body = days_and_total_ne_count[0][0]
day_text_material = bpy.data.materials.new(name=f"Day Text Material")
day_text_material.diffuse_color = (.528, .176, .0, 1)
day_text.data.materials.append(day_text_material)

# Animate the day, # posts, total # of named entities. This is done via a callback method invoked
# when blender is running the animation to change the body of the text. A text object's body content is
# not keyframeable like other object attributes (position, etc) hence the need for this callback method
def update_day_text(scene):
    frame = bpy.context.scene.frame_current
    day_index = int(frame / (frames_to_animate + frames_to_pause))
    current_day, current_ne_count = days_and_total_ne_count[day_index]
    current_post_count = int(day_to_post_count[current_day])
    day_text.data.body = f"{current_day}\n# Posts: {current_post_count}\nTotal # NEs: {current_ne_count}"
    if frame % (frames_to_animate + frames_to_pause) > frames_to_pause and day_index < len(days_and_total_ne_count) - 1:
        # We are in the frame range corresponding to animation (not pausing)
        frame_in_animate = frame % (frames_to_animate + frames_to_pause) - frames_to_pause
        next_day, next_ne_count = days_and_total_ne_count[day_index+1]
        num_days_to_next = (date.fromisoformat(next_day) - date.fromisoformat(current_day)).days
        days_to_add = int(num_days_to_next * (frame_in_animate / frames_to_animate))
        day_in_frame = date.fromisoformat(current_day) + timedelta(days=days_to_add)
        ne_count_to_add = int((next_ne_count - current_ne_count) * (frame_in_animate / frames_to_animate))
        next_post_count = int(day_to_post_count[next_day])
        post_count_to_add = int((next_post_count - current_post_count) * (frame_in_animate / frames_to_animate))
        day_text.data.body = f"{day_in_frame}\n# Posts: {current_post_count + post_count_to_add}\nTotal # NEs: {current_ne_count + ne_count_to_add}"
bpy.app.handlers.frame_change_post.append(update_day_text)

# Finally set the ending frame to the last frame of the animation so the whole video will be rendered
# and a location to render to
bpy.context.scene.frame_end = cur_frame
bpy.context.scene.render.filepath = "./Animation/"

```

Once the script has run the animation will be complete with camera, lighting, animation and ready to render. Blender also has video editing capability which I used to include a snippet of music I wrote a long time ago. The composition is a waltz, in 3/4 time, which corresponds to the 3 "beats" of the animation (puase for 2 seconds, animate transition to next day for 1 second). It's important to add a bit of sonic bling to spice up the experience. Note I only spent a day learning and developing the Blender script above. Even though it appears 2D it is rendered in 3D space and the full power of Blender is available in the scripting interface. This video is a simple proof of concept, far more interesting visualizations are possible.