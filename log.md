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
