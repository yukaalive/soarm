- リーダーアーム(黒いケーブルつけた)
  - The port of this MotorsBus is '/dev/tty.usbmodem5B7B0091361' 
- フォロワーアーム
  - The port of this MotorsBus is '/dev/tty.usbmodem5B7B0098751'
  - 
% lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/tty.usbmodem5B7B0098751 \
  --robot.id=my_awesome_follower_arm \
  --robot.max_relative_target=30 \
  --teleop.type=so101_leader \
  --teleop.port=/dev/tty.usbmodem5B7B0091361 \
  --teleop.id=my_awesome_leader_arm

lerobot-calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem5B7B0091361 \
    --teleop.id=my_awesome_leader_arm

    
lerobot-calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5B7B0098751 \
    --robot.id=my_awesome_follower_arm


lerobot-record --robot.type=so101_follower --robot.port=/dev/tty.usbmodem5B7B0098751 --robot.id=my_awesome_follower_arm --robot.cameras="{wrist: {type: opencv, index_or_path: 0, width: 640, height: 360, fps: 30}, top: {type: opencv, index_or_path: 1, width: 640, height: 360, fps: 30}, side: {type: opencv, index_or_path: 2, width: 640, height: 360, fps: 30}}" --teleop.type=so101_leader --teleop.port=/dev/tty.usbmodem5B7B0091361 --teleop.id=my_awesome_leader_arm --display_data=true --dataset.push_to_hub=False --dataset.private=True --dataset.repo_id=yukaalive/record-test --dataset.single_task="Pick up the plush toy and place it in the pink donut." --dataset.episode_time_s=60 --dataset.reset_time_s=30 --dataset.num_episodes=3 --dataset.fps=30 --dataset.video=true --play_sounds=true
