---
id: aqmxcfbm
title: How does feed function work?
file_version: 1.1.3
app_version: 1.18.31
---

This code snippet is a function called `feed` that handles a GET request to the `/feed` endpoint. It takes in three parameters: `cookie`, `uid`, and `params`. Inside the function, it retrieves a `site_config` from the database and tries to get a `claim` based on the `cookie` and `site_config`. It then checks if the `claim` matches the `uid` parameter. If it doesn't, it sets `read` to `true` and retrieves the `username` of the corresponding `user` from the database.

The function then initializes a `map` and a `feed_ids` vector. It also initializes a `folders`<swm-token data-swm-token=":src/controller/feed.rs:111:1:1:`    folders: BTreeMap&lt;String, Vec&lt;OutFeed&gt;&gt;,`"/> vector and a `feed_id_folder` HashMap. It retrieves folders associated with the `uid` from the database and populates the `folders` vector and `feed_id_folder` HashMap with the results.

Next, it checks if the `username` is not `None` and if the current `feed` is not public, it skips to the next iteration. Otherwise, it creates an `OutFeed` object and adds it to the corresponding entry in the `map` using the folder name as the key.

If an `active_folder` parameter is provided, it checks if it matches the current `feed` folder and if it's not empty. If it doesn't match, it skips to the next iteration. If an `active_feed` parameter is provided and it's not equal to 0, it checks if it matches the current `feed` feed\_id. If it doesn't match, it skips to the next iteration. If it does match, it sets `active_folder` to the current `feed` folder.

Finally, it adds the current `feed` feed\_id to the `feed_ids` vector.

The code snippet ends abruptly, so it's unclear what the remaining code does.
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 src/controller/feed.rs
```renderscript
173    /// `GET /feed`
174    pub(crate) async fn feed(
175        cookie: Option<TypedHeader<Cookie>>,
176        Path(uid): Path<u32>,
177        Query(params): Query<ParamsFeed>,
178    ) -> Result<impl IntoResponse, AppError> {
179        // Get the site config
180        let site_config = SiteConfig::get(&DB)?;
181        let claim = cookie.and_then(|cookie| Claim::get(&DB, &cookie, &site_config));
182        let mut read = false;
183        let username = match claim {
184            Some(ref claim) if claim.uid == uid => None,
185            _ => {
186                read = true;
187                let user: User = get_one(&DB, "users", uid)?;
188                Some(user.username)
189            }
190        };
191    
192        let mut map = BTreeMap::new();
193        let mut feed_ids = vec![];
194    
195        let mut folders = vec![];
196        let mut feed_id_folder = HashMap::new();
197        for i in DB.open_tree("user_folders")?.scan_prefix(u32_to_ivec(uid)) {
198            let (k, v) = i?;
199            let feed_id = u8_slice_to_u32(&k[(k.len() - 4)..]);
200            let folder = String::from_utf8_lossy(&k[4..(k.len() - 4)]).to_string();
201            feed_id_folder.insert(feed_id, folder.clone());
202            let is_public = v[0] == 1;
203            folders.push(Folder {
204                folder,
205                feed_id,
206                is_public,
207            })
208        }
209    
210        let mut active_folder = params.active_folder;
211    
212        for feed in folders {
213            if username.is_some() && !feed.is_public {
214                continue;
215            }
216    
217            let e: &mut Vec<OutFeed> = map.entry(feed.folder.clone()).or_default();
218            let out_feed = OutFeed::new(&DB, feed.feed_id, feed.is_public)?;
219            e.push(out_feed);
220    
221            if let Some(ref active_folder) = active_folder {
222                if active_folder != &feed.folder && !active_folder.is_empty() {
223                    continue;
224                }
225            }
226    
227            if let Some(active_feed) = params.active_feed {
228                if active_feed != 0 {
229                    if active_feed != feed.feed_id {
230                        continue;
231                    }
232                    active_folder = Some(feed.folder)
233                }
234            }
235    
236            feed_ids.push(feed.feed_id);
237        }
238    
239        let mut item_ids = vec![];
240        for id in feed_ids {
241            let mut ids = get_item_ids_and_ts(&DB, "feed_items", id)?;
242            item_ids.append(&mut ids);
243        }
244    
245        let mut read_ids = HashSet::new();
246        let mut star_ids = vec![];
247        let mut star_ids_set = HashSet::new();
248        if let Some(ref claim) = claim {
249            star_ids = get_item_ids_and_ts(&DB, "star", claim.uid)?;
250            star_ids_set = star_ids.iter().map(|(i, _)| *i).collect();
251    
252            read_ids = get_ids_by_prefix(&DB, "read", u32_to_ivec(claim.uid), None)?
253                .into_iter()
254                .collect();
255        }
256    
257        if let Some(ref filter) = &params.filter {
258            if filter == "star" {
259                if active_folder.is_some() {
260                    item_ids.retain(|(i, _)| star_ids_set.contains(i));
261                } else {
262                    item_ids = star_ids;
263                }
264            } else if filter == "unread" {
265                item_ids.retain(|(i, _)| !read_ids.contains(i));
266            }
267        }
268    
269        item_ids.sort_unstable_by(|a, b| a.1.cmp(&b.1));
270        let n = site_config.per_page;
271        let anchor = params.anchor.unwrap_or(0);
272        let is_desc = params.is_desc.unwrap_or(true);
273        let page_params = ParamsPage { anchor, n, is_desc };
274        let (start, end) = get_range(item_ids.len(), &page_params);
275        item_ids = item_ids[start - 1..end].to_vec();
276        if is_desc {
277            item_ids.reverse();
278        }
279        let mut items = Vec::with_capacity(n);
280        for (i, _) in item_ids {
281            let item: Item = get_one(&DB, "items", i)?;
282            let mut is_read = read;
283            if read_ids.contains(&i) {
284                is_read = true;
285            }
286    
287            let is_starred = star_ids_set.contains(&i);
288            let feed_id = get_feed_id(i)?;
289            let folder = if let Some(r) = feed_id_folder.get(&feed_id) {
290                r.to_owned()
291            } else {
292                "".to_owned()
293            };
294            let out_item = OutItem {
295                item_id: i,
296                title: item.title,
297                folder,
298                feed_id,
299                feed_title: item.feed_title,
300                updated: ts_to_date(item.updated),
301                is_starred,
302                is_read,
303            };
304            items.push(out_item);
305        }
306    
307        let has_unread = if let Some(ref claim) = claim {
308            User::has_unread(&DB, claim.uid)?
309        } else {
310            false
311        };
312        let page_data = PageData::new("Feed", &site_config, claim, has_unread);
313        let page_feed = PageFeed {
314            page_data,
315            folders: map,
316            items,
317            filter: params.filter,
318            n,
319            anchor,
320            is_desc,
321            uid,
322            username,
323            active_feed: params.active_feed.unwrap_or_default(),
324            active_folder: active_folder.unwrap_or_default(),
325        };
326    
327        Ok(into_response(&page_feed))
328    }
```

<br/>

This code defines a struct `FormInn` that represents form data for creating or editing a page. It includes fields for `inn_name`, `about`, `description`, `topics`, `inn_type`, `early_birds`, and `limit_edit_seconds`. The `FormInn` struct is derived from the `Deserialize` and `Validate` traits, allowing it to be deserialized from form data and validated. The fields have validation annotations specifying their length constraints.
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 src/controller/inn.rs
```renderscript
121    /// Form data: `/mod/:iid` inn create/edit page
122    #[derive(Deserialize, Validate)]
123    pub(crate) struct FormInn {
124        #[validate(length(min = 1, max = 64))]
125        inn_name: String,
126        #[validate(length(min = 1, max = 512))]
127        about: String,
128        #[validate(length(min = 1, max = 65535))]
129        description: String,
130        #[validate(length(min = 1, max = 128))]
131        topics: String,
132        inn_type: String,
133        early_birds: u32,
134        limit_edit_seconds: u32,
135    }
136    
```

<br/>

This file was generated by Swimm. [Click here to view it in the app](https://app.swimm.io/repos/Z2l0aHViJTNBJTNBZnJlZWRpdCUzQSUzQW1laHJhbi1wcnM=/docs/aqmxcfbm).
