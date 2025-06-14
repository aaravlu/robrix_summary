#### PortalList 索引编号
`PortalList` 在 makepad 中是一个及其复杂的组件
当前我们的 `RoomsList` 只是一个一维的List, 但是在Robrix中, 我们为它手动却抽象出了伪二维的概念.

目前 `RoomsList` 里有3个组, 分别是invited, direct, regular, 把某个组放在最上边(PortalList),
可以很轻易的知道这个组的header索引号, 第一个room索引号, 最后一个room的后一个项目的索引号:

```rust
struct RoomCategoryIndexes {
    /// The index of this room category's header, at which a `<CollapsibleHeader>` widget is displayed.
    header_index: Option<usize>,
    /// The index of the first room in this category that appears immediately after the header.
    first_room_index: usize,
    /// The index after the last room in this category, which is where the next category should start.
    after_rooms_index: usize,
}
```


有了第一个组的索引, 我们就可以很轻易地计算出后边组的索引, 计算新的下一个组的索引, 需要上一个组的数据:
```rust
    fn calculate_indexes(&self) -> (RoomCategoryIndexes, RoomCategoryIndexes, RoomCategoryIndexes) {
        // Based on the various displayed room lists and is_expanded state of each room header,
        // calculate the indices in the PortalList where the headers and rooms should be drawn.
        let should_show_invited_rooms_header = !self.displayed_invited_rooms.is_empty();
        let should_show_direct_rooms_header = !self.displayed_direct_rooms.is_empty();
        let should_show_regular_rooms_header = !self.displayed_regular_rooms.is_empty();

        let index_of_invited_rooms_header = should_show_invited_rooms_header.then_some(0);
        let index_of_first_invited_room = should_show_invited_rooms_header as usize;
        let index_after_invited_rooms = index_of_first_invited_room +
            if self.is_invited_rooms_header_expanded {
                self.displayed_invited_rooms.len()
            } else {
                0
            };

        let index_of_direct_rooms_header = should_show_direct_rooms_header
            .then_some(index_after_invited_rooms);
        let index_of_first_direct_room = index_after_invited_rooms +
            should_show_direct_rooms_header as usize;
        let index_after_direct_rooms = index_of_first_direct_room +
            if self.is_direct_rooms_header_expanded {
                self.displayed_direct_rooms.len()
            } else {
                0
            };

        let index_of_regular_rooms_header = should_show_regular_rooms_header
            .then_some(index_after_direct_rooms);
        let index_of_first_regular_room = index_after_direct_rooms +
            should_show_regular_rooms_header as usize;
        let index_after_regular_rooms = index_of_first_regular_room +
            if self.is_regular_rooms_header_expanded {
                self.displayed_regular_rooms.len()
            } else {
                0
            };

        let invited_rooms_indexes = RoomCategoryIndexes {
            header_index: index_of_invited_rooms_header,
            first_room_index: index_of_first_invited_room,
            after_rooms_index: index_after_invited_rooms,
        };
        let direct_rooms_indexes = RoomCategoryIndexes {
            header_index: index_of_direct_rooms_header,
            first_room_index: index_of_first_direct_room,
            after_rooms_index: index_after_direct_rooms,
        };
        let regular_rooms_indexes = RoomCategoryIndexes {
            header_index: index_of_regular_rooms_header,
            first_room_index: index_of_first_regular_room,
            after_rooms_index: index_after_regular_rooms,
        };
        (
            invited_rooms_indexes,
            direct_rooms_indexes,
            regular_rooms_indexes,
        )
    }
```

##### makepad 对于 SVG 的处理:

我在做 [Display verification status as a badge atop the user profile icon](https://github.com/project-robius/robrix/pull/244)
的时候, 发现 makepad 对于 SVG 的处理并不是很完善, 尤其是居中.

SVG 的本质是 XAML 文档, 这是一种及其复杂的格式, 一个 SVG 文件大小 可能有3KB, 内部属性值可能有几百个, makepad 不可能全部对接,
比如说居中, 我使用主流的图片查看器查看svg都是居中的, 但是在makepad, 里有些 SVG 不能默认居中, 需要调整格式才能完美居中:

```rust
VerificationIcon = <Icon> {
    margin: {left: 0, right: 3, top: 2, bottom: 0}
}
```
注意, 不同的 SVG 可能需要不同的 `margin`, 这取决于 SVG 文件的内部属性, 开发者在外边包一层<View>, 再使用debug:

```rust
<View> {
    debug: true
    // 必须使用align居中<View>内部的组件
    align: {x: 0.5, y: 0.5}
    <Icon> {
        ...
    }
}
```


#### 音频播放
我们肯定不想在同一时间播放两段音频, 所以要有一个 `AudioController`, 而且一定要是全局唯一的.
这是一个后台组件, 负责音频的播放, 暂停, 停止, 所以无需在 UI 上多写任何代码, 它也是不可见的:
```rust
live_design! {
    use link::theme::*;
    use link::shaders::*;
    use link::widgets::*;

    use crate::shared::styles::*;
    use crate::shared::icon_button::*;

    // width & height is 0 because it is just a template.
    pub AudioController = {{AudioController}} {
        width: 0., height: 0.,
        visible: false,
    }
}
```

在任意组件中的handle_startup中使用cx.audio_output即可, 它是一个回调函数, 只需要每次修改output_buffer为目标数据即可,
实际播放逻辑:
```rust
cx.audio_output(0, move |_audio_info, output_buffer|{
    // Modify `output_buffer` per loop
}
```

makepad 默认单声道播放, 如果希望使用双声道, 这么写:
```rust
// left 和 right 都是 `&mut [f32]` 类型的
let (left, right) = output_buffer.stereo_mut();
```

一个简单的例子:
```rust
#[derive(Debug, Clone, Default)]
pub struct Audio {
    pub data: Arc<[u8]>,
    // 声道数
    pub channels: u16,
    // 位深 (通常是16, 24, 32)
    pub bit_depth: u16
}

#[derive(Debug, Clone, Default)]
pub enum Selected {
    // 当前 AudioController 选中了一个音频文件, 并且正在播放此音频
    Playing(TimelineEventItemId, usize),
    // 当前 AudioController 选中了一个音频文件, 并且音频处于暂停状态
    Paused(TimelineEventItemId, usize),
    // 当前 AudioController 什么都没选中
    #[default]
    None,
}

#[derive(Live, LiveHook, Widget)]
pub struct AudioController {
    #[deref] view: View,
    // 跨线程通信, 需要智能指针加锁
    #[rust] audio: Arc<Mutex<Audio>>,
    // 跨线程通信, 需要智能指针加锁
    #[rust] selected: Arc<Mutex<Selected>>,
}

fn handle_startup(&mut self, cx: &mut Cx) {
    let audio = self.audio.clone();
    let selected = self.selected.clone();
    cx.audio_output(0, move |_audio_info, output_buffer|{
        let mut selected_mg = selected.lock().unwrap();
        let audio_mg = audio.lock().unwrap();
        if let Selected::Playing(ref id, mut pos) = selected_mg.clone() {
            let audio = audio_mg.clone();
            let audio_data_len = audio.data.len();
            match (audio.channels, audio.bit_depth) {
                (2, 16) => {
                    // stereo 16bit
                    output_buffer.zero();
                    let (left, right) = output_buffer.stereo_mut();
                    for i in 0..left.len() {
                        if pos + 4 < audio_data_len {
                            // 2^4 = 16, 所以 pos 加4个数作为索引, 分别是0, 1, 2, 3.
                            let left_i16 = i16::from_le_bytes([audio.data[pos], audio.data[pos + 1]]);
                            let right_i16 = i16::from_le_bytes([audio.data[pos + 2], audio.data[pos + 3]]);
                            left[i] = left_i16 as f32 / i16::MAX as f32;
                            right[i] = right_i16 as f32 / i16::MAX as f32;
                            pos += 4;
                            *selected_mg = Selected::Playing(id.clone(), pos);
                        } else {
                            break;
                        }
                    }
                }
                (2, 24) => {
                    // stereo 24bit
                    output_buffer.zero();
                        let (left, right) = output_buffer.stereo_mut();
                        for i in 0..left.len() {
                            if pos + 5 < audio_data_len {
                                // 我们只需要24, 但是 2^5 = 32, 所以第一个留空, pos 加6个数作为索引, 分别是0, 1, 2, 3, 4, 5.
                                let left_i32 = i32::from_le_bytes([0, audio.data[pos], audio.data[pos + 1], audio.data[pos + 2]]);
                                let right_i32 = i32::from_le_bytes([0, audio.data[pos + 3], audio.data[pos + 4], audio.data[pos + 5]]);
                                left[i] = left_i32 as f32 / i32::MAX as f32;
                                right[i] = right_i32 as f32 / i32::MAX as f32;
                                pos += 6;
                                *selected_mg = Selected::Playing(id.clone(), pos);
                            } else {
                                break;
                            }
                        }
                }
                (2, 32) => {
                    // stereo 32bit
                    output_buffer.zero();
                        let (left, right) = output_buffer.stereo_mut();
                        for i in 0..left.len() {
                            if pos + 6 < audio_data_len {
                                // 2^5 = 32, 所以 pos 加8个数, 分别是0, 1, 2, 3, 4, 5, 6 ,7.
                                let left_i32 = i32::from_le_bytes([audio.data[pos], audio.data[pos + 1], audio.data[pos + 2], audio.data[pos + 3]]);
                                let right_i32 = i32::from_le_bytes([audio.data[pos + 4], audio.data[pos + 5], audio.data[pos + 6], audio.data[pos + 7]]);
                                left[i] = left_i32 as f32 / i32::MAX as f32;
                                right[i] = right_i32 as f32 / i32::MAX as f32;
                                pos += 8;
                                *selected_mg = Selected::Playing(id.clone(), pos);
                            } else {
                                break;
                            }
                        }
                }
                _ => {
                    // 其他格式挺稀有的, 先不管了.
                }
            }

            // Use `pos + audio.bit_depth.ilog2()` rather than `pos` to ensure no panic when computing isize mentioned above.
            if pos + audio.bit_depth.ilog2() as usize > audio_data_len - 1 {
                if let Some((_audio, _old_pos, old_playing_status)) = AUDIO_SET.read().unwrap().get(id) {
                    *old_playing_status.lock().unwrap() = false;
                }
                Cx::post_action(AudioControllerAction::UiToPause(id.clone()));
                *selected_mg = Selected::None;
            }
        } else {
            output_buffer.zero();
        }
    });
}

// 传统的音频控件播放逻辑, 简单来说就是当新的播放通知发来的时候, 先看看当前 `AudioController` 自身的状态,
// 看看要不要保存当前的音频状态, 再比对 `AudioController` 当前状态和新的播放通知的目标状态, 去修改哪些数据.
fn handle_actions(&mut self, cx: &mut Cx, actions: &Actions) {
    for action in actions {
        match action.downcast_ref() {
            Some(AudioMessageUIAction::Play(new_id)) => {
                let selected = self.selected.lock().unwrap().clone();
                match selected {
                    Selected::Playing(current_id, current_pos) =>  {
                        if current_id != new_id.clone() {
                            cx.action(AudioControllerAction::UiToPause(current_id.clone()));
                            restore_audio_play_status(&current_id, current_pos);
                            self.switch_to_audio(new_id);
                        }
                    }
                    Selected::Paused(current_id, current_pos) => {
                        if &current_id == new_id {
                            *self.selected.lock().unwrap() = Selected::Playing(new_id.clone(), current_pos);
                        } else {
                            restore_audio_play_status(&current_id, current_pos);
                            self.switch_to_audio(new_id);
                        }
                    }
                    Selected::None => {
                        self.switch_to_audio(new_id);
                    }
                }
            }
            Some(AudioMessageUIAction::Pause(new_id)) => {
                let mut selected_mg = self.selected.lock().unwrap();
                if let Selected::Playing(current_id, current_pos) = selected_mg.clone() {
                    if &current_id == new_id {
                        restore_audio_play_status(&current_id, current_pos);
                        *selected_mg = Selected::Paused(new_id.clone(), current_pos);
                    }
                }
            }
            Some(AudioMessageUIAction::Stop(new_id)) => {
                let mut selected_mg = self.selected.lock().unwrap();
                match selected_mg.clone() {
                    Selected::Playing(current_id, _current_pos) | Selected::Paused(current_id, _current_pos) => {
                        restore_audio_play_status(new_id, WAV_HEADER_SIZE);
                        if &current_id == new_id {
                            *selected_mg = Selected::None;
                        }
                    }
                    _ => { }
                }
            }
            _ => { }
        }
    }
}
```
