## Storm Tasks

Storm Task is a runnable not a thread. Multiple tasks could be run in a single-thread executor and they are scheduled by the incoming messages. 
Storm abstracted unbounded data as stream, which is produced by `Spout` and processed by `Bolt`. `Spout` and `Bolt` form into a topology by grouping them together. `Spout` and `Bolt` are serilizable, created at client side, submitted to nimbus, assigned to workers and ran by executors. 


![storm_threads](https://lh3.googleusercontent.com/inObeBBqpVMQcaKk6T8teWHsJV_vQxjZ7YRoY88gMnqsnWVtOJ252E6YoFx2tyPeGellBIOaOflBE0EaBylmLL3AZDPBg8wJx4S9Elx4g6ZJ-MUhUHciL8jrtLIfBvYLEQi9sBzkNeAp5eyWLYXe4Wq1HMjz_xMv-uX53S6VacYJ3vFQVGYUm5iXpqoa2Cd8H05cVn8J2VdVo1T_c4A8x1iU9mp8s6mmNdjDNIOJRn79WrRdcCU7ay7YC_FyRNAwcViQsx9_xfhruu8l6Xo1mgKejk4Ar0WezqH_HwGUjiqKM_R_8a23s_5tZ6kXepcsS2cWXaPXyj_3V7muREcj9ZIp05j7sMWmWxPiFmpAVIeDVjnmF0LBlP3W7mUhd1lIelVjV8MgjICqyFG3tKqHO7DGuf99RX7ow6p1LjbUNXaUbUff7eJzqqfNnii928mm0hPf4ftdvyeNKYqD5M4K-m-72bspF9zy_AuNZrZu23FO8NEjeIHrVAv-3S3xzpHgf2TFusk6vypmNmSFyGVtTWMHFAxjhe3ncdP3t29UgQM=w719-h761-no)

## Fault tolerance

Messages could be lost or reordered when sent from one node to another. Storm guarantees that a message is processed at least once. 