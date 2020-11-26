ALTER table app_component add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table app_create_page add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table app_hot_search add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table app_module_product add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table app_start_page add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table app_window add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';



ALTER table t_at_quality_recommend add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table t_at_marketing_category add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table t_at_marketing_page add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';



ALTER table t_pm_coupon_details add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table t_pm_coupon_package add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table t_pm_coupon_spu_white_list add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table t_pm_user_coupon add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';

 ALTER TABLE t_pm_coupon_package drop key `uni_package_id`;

ALTER TABLE t_pm_coupon_package add UNIQUE key `uni_package_id_type` (`coupon_package_id`, app_type) USING BTREE;



ALTER table activity add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table activity_group add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table activity_spu add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';
ALTER table activity_subject add COLUMN `app_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'APP类型：0、fingo 1、meOne';

INSERT INTO `market_activity`.`activity_template`(`id`, `name`, `image`, `type`, `description`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (8, '直播', NULL, 8, NULL, '2020-11-12 09:52:22', 0, '2020-11-12 09:52:22', 0);

INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (96, 8, '使用国家', NULL, 1, '', '%s == <p0>', 53, '2020-09-01 09:32:34', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (97, 8, '个人本次活动每单起购数', NULL, 1, '', '', 150, '2020-09-01 09:38:51', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (98, 8, '个人本次活动每单限购数', NULL, 1, '', '', 151, '2020-09-01 09:39:12', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (99, 8, '个人本次活动总限购数', NULL, 1, '', '', 152, '2020-09-01 09:39:32', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (100, 8, '单日限购', NULL, 1, '', '', 153, '2020-09-01 09:39:59', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (101, 8, '是否预热', NULL, 1, '', '', 4, '2020-09-01 09:41:14', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (102, 8, '活动起止时间', NULL, 1, '<p0> - <p1>', '%s <= <p1>', 50, '2020-09-01 09:53:33', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (103, 8, '预热开始时间', NULL, 1, '', '%s >= <p0>', 51, '2020-09-01 09:54:45', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (104, 8, '是否存在商品元素', NULL, 1, '', '', 56, '2020-09-01 09:56:12', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (105, 8, '是否含有活动价', NULL, 1, '', '', 5, '2020-09-01 09:58:57', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (106, 8, '促销', NULL, 1, '', '', 100, '2020-09-01 09:59:27', 0, '2020-09-17 10:05:40', NULL);
INSERT INTO `market_activity`.`activity_template_element`(`id`, `template_id`, `name`, `type`, `status`, `value_expression`, `judge_expression`, `order_id`, `create_time`, `creator_id`, `update_time`, `operator_id`) VALUES (107, 8, '直播检验', NULL, 1, '', '', 300, '2020-09-01 09:59:27', 0, '2020-09-17 10:05:40', NULL);